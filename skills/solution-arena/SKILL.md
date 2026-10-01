---
name: solution-arena
description: Run several independent solver subagents on the identical task, then rank their solutions with a blind judge, the orchestrator, or a pairwise tournament. Use when a coding task has multiple valid approaches (ambiguous bugs, refactors, optimizations, design trade-offs) and you want competing candidates instead of trusting one agent path.
disable-model-invocation: true
metadata:
  origin: "community - adapted from irr/arena-mode (MIT, Ivan Rocha)"
---

# Solution Arena

Best-of-N generation for coding work. N solvers attack the same locked task
in isolation, every candidate is verified, and a ranking picks the winner.
Nothing is applied until the user chooses.

The core insight: one agent's first approach is one sample from a wide
distribution. When several approaches are plausible, sampling a few in
parallel and comparing them is cheaper than discovering late that the first
path was the wrong one.

This skill is invoked explicitly (`/solution-arena ...`) because it spends
N+1 subagent runs. It does not auto-activate.

## When to Activate

Use solution-arena when:
- a bug has several plausible root causes and the fix depends on which one is real
- a refactor or optimization can be done in materially different ways
- the user asks for competing implementations, alternatives, or "best of N"
- a high-cost change deserves comparison before committing to one path

Do NOT use it for:

| Instead of solution-arena | Use |
| --- | --- |
| A go / no-go or strategy decision with no code to compare | `council` |
| Checking that one finished output is correct | `santa-method` or `verification-loop` |
| Splitting a large feature into parallel workstreams | `dmux-workflows` |
| A task with one obvious approach | just do it |

## Invocation

```text
/solution-arena "<task>" [n=<3-6>] [strategy=judge|main|tournament] [solver-model=<alias>] [judge-model=<alias>] [model=<alias>]
```

| Argument | Default | Meaning |
| --- | --- | --- |
| task | required | The problem every solver must solve |
| `n=` | `4` | Number of solvers. Clamp to 2..6 and say so if the user asked for more |
| `strategy=` | `judge` | How candidates are ranked (see below) |
| `solver-model=` | session model | Model for every solver |
| `judge-model=` | session model | Model for the judge (`judge` and `tournament` only) |
| `model=` | session model | Sets both roles; the role-specific flags override it |

Model values are subagent aliases only: `opus`, `sonnet`, `haiku`, `fable`.
Anything else falls back to the session model with a one-line warning.

Before spawning anything, echo the parsed values so the user can stop a
misparse:

```text
Solution arena: n=4, strategy=judge, solvers=sonnet, judge=opus
Task: Refactor the auth middleware to accept JWT and API keys without breaking existing clients
```

## How It Works

### 1. Lock the task

Write one task prompt and send it, byte-identical, to every solver. Include:
- the user's task verbatim
- relevant file paths and constraints you already know (save each solver the same discovery)
- the verification command(s) the solver must run (tests, typecheck, lint)
- the required return format (below)

Then give each solver one short **distinct lens** appended after the locked
prompt, so the candidates actually differ. Without a lens, N solvers from one
model tend to converge on the same answer and the arena buys nothing.

| Lens | Appended line |
| --- | --- |
| minimal | Prefer the smallest diff that fully solves the task. |
| structural | Prefer the cleanest long-term structure, even if the diff is larger. |
| defensive | Prefer the approach with the fewest new failure modes and the best test coverage. |
| performance | Prefer the approach with the best runtime and resource behavior. |
| conventional | Match existing patterns in this codebase as closely as possible. |
| contrarian | Choose an approach different from the most obvious one, and justify it. |

### 2. Spawn solvers in parallel

Launch all N in one message so they run concurrently. Any solver that edits
files gets its own git worktree (`isolation: "worktree"` on the Agent tool, or
`scripts/orchestrate-worktrees.js`). Solvers never see each other's output.

Each solver returns:

```text
APPROACH: 2-4 sentences
CHANGES: the diff, or the worktree branch name holding it
VERIFICATION: commands run and their pass/fail result, verbatim
CONFIDENCE: 1-5
RISKS: what could break, and what was not verified
```

A candidate with no verification evidence is ranked below every verified
candidate unless the task is design-only.

### 3. Verify independently

Do not trust self-reported results. For each worktree candidate, the
orchestrator re-runs the project's test command itself and records the real
outcome. A candidate that fails the existing test suite is disqualified, not
merely penalized, unless the task was to change that behavior.

### 4. Rank

Before ranking, relabel candidates as Solution A, B, C... in a **shuffled**
order, and strip lens names and model names. Judges show position bias toward
the first candidate; shuffling stops it from always favoring solver 1.

**`judge` (default).** Spawn one judge subagent with the task, the rubric,
the anonymized candidates, and the verification results. It returns a ranking
with a reason per rank. Use when you want generation and evaluation separated;
pairing cheap solvers with a stronger judge is a good cost trade.

**`main`.** The orchestrator ranks against the rubric itself. Cheapest. Use
for small tasks; note that the orchestrator wrote the locked prompt and may
favor candidates that match its own framing.

**`tournament`.** Pair candidates (one bye if N is odd). For each pair, a
critic compares them side by side and names a winner with 2-3 concrete
reasons. Winners advance until one remains; losers are ordered by the round
they fell in. Use for high-stakes design choices, where adversarial
comparison surfaces weaknesses a single pass misses.

### Rubric

| Criterion | Weight |
| --- | --- |
| Correctness and completeness: solves the stated task, verified | High |
| Compatibility: existing behavior, APIs, and tests preserved | High |
| Focus: smallest change that does the job, no drive-by edits | Medium |
| Clarity: readable, fits codebase conventions | Medium |
| Risk surface: fewest new failure modes | Medium |
| Plan adherence, when an agreed plan exists | Medium |

The user can supply a different rubric; use theirs instead.

### 5. Present and stop

```markdown
## Arena results (strategy: judge, n=4)

### Ranking
1. **Solution C**: strategy-pattern auth resolver. Tests pass (212/212).
   - Why first: only candidate that keeps the legacy header path working
2. **Solution A**: inline branch in middleware. Tests pass (212/212).
   - Why second: correct, but duplicates token parsing in two places
3. **Solution D**: new auth service. 3 tests fail.
   - Why third: breaks session refresh in `auth.test.ts`

### Key differences
- C and A agree on token parsing; they differ on where dispatch lives
- D is the only candidate that changes the public interface

### Recommendation
Apply **Solution C**: verified, non-breaking, smallest surface of the passing two.

Which one should I apply?
```

Wait for the user's choice. Then merge only the chosen worktree's changes
into the working branch and remove the other worktrees. Never commit a
candidate on your own.

## Examples

```text
/solution-arena "Fix the intermittent timeout in tests/api/upload.test.js; find the real cause, do not raise the timeout"
```
Default judge, four lenses. Useful because flaky-test causes are usually
contested; candidates that only bump the timeout fail the lens and the rubric.

```text
/solution-arena "Cut p95 latency of GET /reports by half without changing the response shape" n=3 strategy=main solver-model=sonnet
```
Three cheaper solvers, orchestrator ranks. Ask solvers to include before/after
timings in VERIFICATION.

```text
/solution-arena "Propose how to split billing out of the monolith. Approaches and trade-offs only, no code" n=4 strategy=tournament judge-model=opus
```
Design-only. Solvers skip worktrees; the tournament stress-tests each proposal.

## Anti-Patterns

- **Identical solvers.** Same model, same prompt, no lens: four copies of one answer at four times the cost.
- **Shared workspace.** Solvers editing the same checkout overwrite each other and the comparison is meaningless.
- **Ranking on claims.** Taking "all tests pass" from a solver without re-running them.
- **Leaking identity.** Showing the judge which model or lens produced a candidate, or always listing solver 1 first.
- **Auto-apply.** Merging the top-ranked candidate without the user's choice.
- **Arena for everything.** Running N+1 agents on a one-line fix. If one approach is obvious, skip the arena.

## Best Practices

- Keep N between 3 and 5. Gains flatten quickly and cost grows linearly.
- Put the verification command in the locked prompt; it is the cheapest way to make candidates comparable.
- For refactors, state the compatibility requirement in the task text, not only in the rubric.
- If the top two candidates are close, say so and show the diff between them rather than inventing a margin.
- If every candidate fails verification, report that and ask before running a second round.

## Related Skills

- `council`: structured disagreement for decisions without code to compare
- `santa-method`: dual independent review of a single output
- `verification-loop`: the checks each candidate should pass
- `dmux-workflows`: parallel agents on different subtasks rather than the same one
