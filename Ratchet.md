---
name: ratchet
description: Run an iterative build-verify loop where a builder subagent implements something and one or more verifier checks (structural, numeric, logic) determine whether it's done. Findings only move forward through a fixed lifecycle and never get re-litigated once truly closed — like a ratchet that can't slip back. Use whenever the user wants to "loop until it passes," iteratively fix and re-check a feature/script/model, or asks for an agentic build-test-fix cycle with a hard iteration cap, regression protection, and rollback on regression. Especially relevant when verification has both an objective/measurable component (tests, metrics, dependency impact) and a subjective/judgment component (does this still make sense, did this introduce a subtle bug).
---

# Ratchet

A controlled build-verify loop. A builder implements, verifiers check the result against distinct criteria, and findings move forward through a one-way lifecycle — never backward, never re-litigated once truly closed. State (iteration count, findings, scores, checkpoints) lives outside any agent so nothing depends on an agent "remembering" what's already happened.

## Core principle

Every check in a verify step falls into one of three kinds. Route each to the right place — never let an agent do a check that could be code, and never let code attempt a check that requires judgment.

| Check type | What it answers | Who/what does it | Example |
|---|---|---|---|
| **Structural** | What does this change actually touch? | A deterministic tool, with a fallback ladder (below) | call graph, downstream dependents, what broke vs. what's now orphaned |
| **Numeric / computable** | Does this pass a measurable threshold, and is it still improving? | A script, never a subagent | test suite pass/fail, Sharpe/drawdown/accuracy vs. a defined threshold |
| **Judgment / logic** | Given the facts above, does this still make sense, and how confident is that? | A verifier subagent, narrowly scoped | leakage, subtle bugs, wrong intent — anything a script can't define |

If a check can be written as `f(output) → bool`, write it as code. Reserve subagent judgment for genuinely ambiguous calls, and always attach a confidence level to those calls — pass/fail alone isn't enough once more than one verifier is in play.

## Finding schema

Every finding — from any gate — uses this shape. No bare pass/fail strings.

```json
{
  "id": "F-12",
  "severity": "critical | major | minor | cosmetic",
  "owner": "builder",
  "source": "numeric_gate | structural_check | logic_verifier",
  "status": "open | fixed | verified | closed",
  "confidence": 0.92,
  "evidence": "feature computed after train/test split, causing leakage"
}
```

`severity` and `confidence` are what make the arbiter's job possible once you have more than one verifier. A critical, high-confidence finding and a cosmetic, low-confidence one must never be weighted the same — treating them identically is the single biggest way this kind of loop produces noise instead of signal.

## Status lifecycle

Findings move through exactly one direction. This is the actual "ratchet" — not regression detection itself, but the rule that nothing is trusted as resolved until it's survived a second look.

```
Open → Fixed → Verified → Closed
```

- **Open** — identified, not yet addressed.
- **Fixed** — builder addressed it; not yet re-checked.
- **Verified** — the *next* full gate pass (structural + numeric + logic, run in full, not just the originally-failing check) confirms it still holds.
- **Closed** — verified across two consecutive passes, or the user explicitly accepts it. Only `closed` findings are dropped from what gets fed back to the builder.

Why "Fixed" isn't "Closed": regressions are actually caught by the numeric gate re-running the *entire* suite every iteration, not by a special regression-detector — if iteration 2 fixes A but reintroduces B, B shows up as failing again on the very next full pass regardless. What the four-stage lifecycle adds on top of that is trust, not detection: a finding that passed once by luck (flaky test, marginal metric) shouldn't be treated with the same confidence as one that's held for two passes running. `Fixed → Verified` is where that distinction lives.

## Structural check: fallback hierarchy

Don't assume a dependency-graph tool is connected. Degrade in this order, and log which level you actually ran at — a structural check run at "manual review" is weaker evidence than one run at "dependency graph," and the logic verifier and arbiter should know that.

```
1. Dependency graph / code-graph MCP (e.g. codemap, Sourcegraph) — exact call graph, exact diff impact
2. Git diff analysis — what files/lines changed, no semantic understanding of impact
3. AST analysis — parse the changed files directly, find references within them
4. Manual review — logic verifier subagent reads the diff and reasons about impact directly
```

Each level down is cheaper to set up but less reliable. Record the level used in the state file so confidence scoring downstream can account for it.

## Numeric gate: thresholds and convergence

Define thresholds once, before iteration 1 — never improvise mid-loop because an iteration is "close enough."

Binary pass/fail isn't always the right model. Where the metric is continuous and improving slowly (0.901 → 0.904 → 0.905 → 0.906), add convergence detection so the loop doesn't grind out the full iteration cap on diminishing returns:

```
if improvement < ε for N consecutive iterations:
    terminate (status: converged, not failed)
```

Tune ε and N deliberately per task — don't default to an arbitrarily small ε. A flat run of small improvements can also be a plateau right before a larger fix lands; terminating too early there discards progress that was about to pay off. This is a judgment call the user makes explicitly, same as the original pass/fail thresholds.

## Checkpoints and rollback

Without this, an iteration that makes things *worse* just continues silently to the next round. Snapshot state at every iteration and roll back if the score regresses:

```
checkpoints/
    iter1/  (code state + metrics + score)
    iter2/
    iter3/
```

```
if current_score < previous_score:
    rollback to previous checkpoint
    feed the regression itself back to the builder as a new critical finding
    do not simply continue forward from the worse state
```

## Roles

- **Builder subagent** — implements or fixes. Receives only `open` and `fixed`-but-not-yet-`verified` findings, never `closed` ones.
- **Structural check** — runs at the highest available fallback level. Produces facts, not opinions.
- **Numeric gate** — a script. Defines thresholds and convergence parameters up front. Outputs structured results (JSON), including the score used for rollback comparison.
- **Logic verifier subagent** — invoked when the numeric gate fails, or to assess confidence on an ambiguous structural result. Given a narrow question tied to specific evidence, and required to emit a confidence score with every finding, not just a verdict.
- **Arbiter** (once more than one verifier is in play) — applies a pre-written decision rule to the verifiers' findings, weighted by severity and confidence. Does not re-review the work itself; if it starts reading between the lines instead of applying the rule, fix the rule, not the arbiter's behavior.

Past 4 active roles, you're usually subdividing without adding a genuinely new failure class — don't add a role unless you can name one nothing else owns.

**Out of scope, deliberately:** parallel builders working on different modules simultaneously, and the merge-conflict resolution that creates. That's a distinct orchestration problem with its own failure modes (conflicting edits, integration ordering) and belongs in a layer above Ratchet, not folded into it.

## State file

```json
{
  "task": "short description of what's being built",
  "iteration": 1,
  "max_iterations": 5,
  "findings": [
    { "id": "F-1", "severity": "critical", "owner": "builder", "source": "numeric_gate", "status": "open", "confidence": 0.9, "evidence": "..." }
  ],
  "thresholds": { "...": "defined once, before iteration 1" },
  "convergence": { "epsilon": 0.001, "n_consecutive": 2 },
  "scores": { "iter1": 0.71, "iter2": 0.84 },
  "structural_check_level": "dependency_graph | git_diff | ast | manual",
  "checkpoints_dir": "checkpoints/"
}
```

## Workflow

1. **Setup.** Define numeric thresholds, convergence parameters (ε, N), and confirm which structural check level is available. Once, before iteration 1.
2. **Build.** Builder addresses only findings with status `open` or `fixed-not-yet-verified`.
3. **Checkpoint.** Snapshot current state before re-verifying.
4. **Structural check.** Run at the highest available fallback level; log facts and the level used.
5. **Numeric gate.** Run the full suite (not just previously-failing checks). Compute score. Compare against thresholds and convergence criteria.
6. **Rollback check.** If `current_score < previous_score`, roll back to the prior checkpoint and log the regression as a new critical finding. Otherwise continue.
7. **Logic verifier** (if triggered). Narrow the cause, attach a confidence score and evidence.
8. **Arbiter** (if multiple verifiers). Apply the pre-written rule, weighted by severity/confidence.
9. **Update findings.** Move resolved items `open → fixed`. On the *next* full pass, promote surviving `fixed` items to `verified`, then to `closed` after a second consecutive clean pass. Newly found or regressed items go in as new `open` findings.
10. **Narrowing.** The builder's next prompt contains only non-`closed` findings. Closed items are never re-sent.
11. **Stop conditions.** Done when all findings are `closed`. Also stop on convergence (ε/N met) or on hitting `max_iterations` — report full history either way, including any rollbacks.

## Worked example

```
Iteration 1: build feature X
  checkpoint saved
  structural (level: dependency_graph): clean, one integration point
  numeric: score 0.71, 3 metrics fail
  logic verifier: 2 of 3 failures trace to distinct causes, confidence 0.85 and 0.9
    → findings: F-1 [critical, open, causeA], F-2 [major, open, causeB], F-3 [minor, open, causeC]

Iteration 2: builder fixes F-1 and F-2 only
  checkpoint saved
  structural: clean
  numeric: score 0.84 (improved, no rollback) — causeA and causeB metrics now pass
    → F-1, F-2 status: open → fixed
  logic verifier: confirms no regression on the previously-passing checks
    → F-1, F-2 status: fixed → verified (will close after one more clean pass)
    → F-3 still open

Iteration 3: builder fixes F-3
  checkpoint saved
  structural: clean
  numeric: score 0.86 — all metrics pass, improvement now < ε
    → F-1, F-2: verified → closed (second clean pass)
    → F-3: open → fixed
  STOP — convergence criterion met; report F-3 as fixed-but-not-yet-reverified, recommend one more pass if the user wants full closure.
```

## Common mistakes this prevents

- Letting a verifier subagent both run and judge a numeric test.
- Letting independent verifiers see each other's verdict before submitting their own.
- Resending `closed` findings to the builder "for context."
- Tracking iteration count or finding status from an agent's memory instead of the state file.
- Improvising thresholds or ε mid-loop because an iteration is "close enough."
- Treating a `Fixed` finding as done without a second clean pass — this is how regressions slip through silently.
- Letting a worse iteration continue forward unflagged instead of rolling back.
- Collapsing severity and confidence into a single pass/fail bit, which makes arbitration between verifiers impossible.

## When NOT to use this skill

If there's no measurable or structurally-checkable gate at all (pure subjective creative work), this loop adds process without benefit — a single builder + single reviewer pass is enough, and you don't need state tracking, checkpoints, or convergence detection.

If you need multiple builders working in parallel on different modules, this skill's loop still governs each builder's own verify cycle, but the merge/integration layer between them needs separate orchestration not covered here.
