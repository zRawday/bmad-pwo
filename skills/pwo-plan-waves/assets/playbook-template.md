---
title: Parallelization Playbook
produced_by: pwo-plan-waves (S1)
consumed_by: pwo-build-phase0 (S2), pwo-run-wave (S3)
project: {project-name}
planned: {timestamp}
applicability: {GO | NO-GO | pending-human}
width: {real DAG width after adversarial verification}
waves: {count}
critical_path_depth: {n}   # the makespan floor
---

# Parallelization Playbook

> The project instance of the parallel-wave method. S2 builds the Phase 0 spec below; S3 runs each
> wave from its lane cards. This playbook is the single source of truth — regenerate it (re-run S1)
> or hand-edit it; do not let S2/S3 drift from it.

## Provenance & precondition

- **Inputs:** epics `{path}` · architecture `{path}` · project-context `{path}` · mockups `{path}`.
- **Harness-profile (S0) check:** `{overall: clean | resolved}` — {one line: all applicable Facts
  clean, or which escalation was resolved and how}. *(S1 refuses to freeze the mechanism otherwise.)*

## Applicability gate (P1/P2 → GO/NO-GO)

| Criterion | Verdict | Evidence |
| --------- | :-----: | -------- |
| P1 — disciplined architecture | {pass/fail} | {layers/ports/single-source/registries — cite the architecture} |
| P2 — mockups + verified smoke harness | {pass/fail} | {whole-app mockups present; smoke_harness `{value}` verified as a probe} |
| Width ≥ ~4–5 | {pass/fail} | {real width after adversarial DAG = {n}} |
| Large backlog | {pass/fail} | {story count / wall-clock matters} |

**Decision: {GO | NO-GO}.** {On NO-GO: *build sequentially* — the orchestration overhead won't
repay; this is the deliverable, stop here. On GO: one line.} · **Human sign-off:** {name / pending}

## The DAG

```
{the blocked-by DAG: each story and its parents — e.g. ascii or an edge list}
```

- **Critical path (makespan floor):** {the longest chain} — depth {n}.
- **Refuted false edges (width recovered):** {child←parent : why it wasn't real}, …
- **Hidden edges found + neutralization:** {child needs symbol from a later story → hoist contract `X` in Phase 0 / reorder}, …

## Hotspots

| File | Kind | Protocol |
| ---- | ---- | -------- |
| {file} | {flat-registry / structured-registry / contract-test / migration-set / glyph-map / co-composed-screen / shared-store / schema} | {the merge/ownership protocol the lane cards carry} |

## Wave plan

> Each wave closes on a coherent, smokable increment (3-force cut: DAG · coherence-to-smoke · ~10 cap).

### Wave {n} — {label}  · type: {foundation | screen}  · {k} lanes

- **Lanes:** {keys}
- **DAG readiness:** all parents merged before this wave.
- **End-of-wave gate:**
  - *foundation* → **jest + empirical sweep** of {the numeric range to sweep}.
  - *screen* → **smoke-vs-mockups, delegated to S5 (`pwo-ui-smoke`)** with this smoke criterion:
    - **journey:** {ordered steps to play}
    - **reference mockup(s):** {paths}
    - **expected live figures:** {label = value}, …
- **Render-deferred entries (if any):** {lane → its nav entry lands in wave {m}}.

*(repeat per wave)*

## Phase 0 spec (for S2)

- **Migration-set guard** — {what it asserts}; S2 must **prove it RED** by fault injection.
- **Seams to hoist** — for each: {port name} · consumer {key} · **position/ordering constraint:**
  {esp. FK ordering — purge/delete BEFORE the referenced-row delete}.
- **`.gitattributes`** — `merge=union` on {flat registries only}; **never** {state file · structured registry · contract `.test.ts`}.
- **Native deps to pre-provision** — {dep@version}, … (peer/transitive + export surface verified: {note}).
- **Test-ids split per screen** — `test-ids/<screen>.ts` + barrel; the per-screen split: {list}.
- **Worktree workspace + ownership** — `{worktree_workspace}`; lanes never write the coordination file (orchestrator owns it 100%).

## Per-lane cards

### {key} — {title}  · tier: {trivial | standard | critical}

- **Footprint:** {files/folders the lane may touch}
- **Hotspots + protocol:** {file → protocol}, … (or "none — fully disjoint")
- **Entry point** (UI lanes): {nav entry} · {reachable this wave | deferred to wave {m}}
- **⚙ Constraint (verbatim — must land in the story spec):**
  > {the imperative the lane obeys — purity / single-source / don't-touch / hotspot protocol;
  >  point at the canonical type: "follow `<type>`; the model wins where this prose differs"}

*(repeat per story; tier derivation: pure-engine→trivial · compose→standard · money/barème |
FK/cascade | migration | new native dep→critical. Critical lanes run `dev=max` at S3.)*

## Human validation

- **Wave plan validated:** {name / pending} — the wave cut, the critical path, the Phase 0 spec.
- **Load-bearing values cross-checked against the source of truth:** {seeds/barèmes/constants → verified}, …
