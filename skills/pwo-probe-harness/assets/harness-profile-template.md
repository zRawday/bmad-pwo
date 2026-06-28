---
title: Harness Profile
produced_by: pwo-probe-harness (S0)
consumed_by: pwo-plan-waves (S1)
project: {project-name}
probed: {timestamp}
copies_per_probe: {n}
overall: {clean | needs-attention}   # needs-attention if ANY applicable Fact is not clean
---

# Harness Profile

> Empirical ground truth about what this harness allows, established by execution — not
> assumption. S1 refuses to freeze the parallelization mechanism unless every applicable
> Fact below is `clean` (or its escalation has been resolved by the human).

## Verdict summary

| Fact | Question | Verdict | Copies agree? | Determination (one line) |
| ---- | -------- | :-----: | :-----------: | ------------------------ |
| F1 | Can a workflow subagent spawn sub-agents? | {clean/partial/broken} | {yes/no} | {e.g. no Agent tool → nesting impossible → code-review stays top-level} |
| F2 | Does invoking the real Skill leak to the main repo? | {…} | {…} | {e.g. real skill writes to main repo + autocommit fires → lanes must emulate} |
| F3 | Does merge=union silently corrupt a structured registry? | {…} | {…} | {e.g. silent splice on the 2nd appender → split test-ids + tsc backstop} |
| F4 | Does concurrent worktree-add lock/deadlock .git? | {…} | {…} | {e.g. brief lock, bounded retry recovers} |
| F5 | Does a declared dep's real API match the lane's assumption? | {clean/partial/broken/n-a} | {…} | {e.g. fs surface moved to /legacy subpath → verify at integration} |

## Per-Fact detail

For each Fact, record both copies' raw observations and the folded verdict.

### F1 — workflow-subagent toolset & nesting
- **Verdict:** {…}  · **Copies agree:** {…}
- **Determination:** {the factual answer}
- **Copy A observations:** {raw tool list / ToolSearch result / spawn outcome}
- **Copy B observations:** {raw}
- **Implication for S1:** {what this forces in the plan}

### F2 — emulate-vs-invoke (main-repo leak)
- **Verdict:** {…}  · **Copies agree:** {…}  · **Repo restored:** {yes/no}
- **Determination:** {leak path + autocommit observation}
- **Copy A observations:** {raw}
- **Copy B observations:** {raw}
- **Implication for S1:** {emulate vs invoke}

### F3 — merge=union on a structured registry
- **Verdict:** {…}  · **Copies agree:** {…}
- **Determination:** {corrupts / safe + the corruption shape + which appender first triggered it}
- **Copy A observations:** {raw merged bytes / parse result}
- **Copy B observations:** {raw}
- **Implication for S1:** {union policy: split + hand-resolve + backstop, or safe}

### F4 — concurrent worktree add
- **Verdict:** {…}  · **Copies agree:** {…}
- **Determination:** {lock occurred? retry recovered? both worktrees created?}
- **Copy A observations:** {literal lock error + retry outcome}
- **Copy B observations:** {raw}
- **Implication for S1:** {concurrent-cut + retry, or serialize}

### F5 — declared dep's real API surface
- **Applicable:** {yes/no}  · **Dep:** {name or —}
- **Verdict:** {…}  · **Copies agree:** {…}
- **Determination:** {matches / mismatches list}
- **Copy A observations:** {real exports + peers}
- **Copy B observations:** {raw}
- **Implication for S1:** {trust the shim, or verify-at-integration / fix}

## Escalations (Facts not clean)

> One block per non-clean applicable Fact. If all are clean, write "None — all applicable Facts clean."

### {Fact id} — {clean-reason missing}
- **What was observed:** {the disagreement or the broken/partial state}
- **Why it blocks the plan:** {what S1 cannot safely assume}
- **Options presented to the human:** {re-probe ×2 · manual investigation · accept-with-documented-risk · adapt the mechanism}
- **Human decision:** {recorded after escalation}
