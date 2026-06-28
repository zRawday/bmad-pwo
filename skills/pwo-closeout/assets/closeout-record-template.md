---
title: Closeout Record
produced_by: pwo-closeout (S4)
consumed_by: the human · bmad-retrospective (follows) · the maturation report (sibling artifact)
project: {project-name}
ran: {timestamp}
baseline_main_head: {sha main was at when closeout began}
final_main_head: {sha after the consolidation merged}
gate_cleared: {true | false}   # the E2E campaign — true only when every journey PASSED (or its findings were hotfixed + re-smoked green)
overall: {validated | blocked | pending-human}   # validated only when gate cleared + consolidation green + human signed off
---

# Closeout Record — {project-name}

> The end-of-backlog **bookend symmetric to Phase 0**: the project validated E2E + the parallelization
> debt paid down. Everything here is verified against **real git / real output / S5's verdict**, not a
> self-report. The durable, reinjectable **maturation report** is the sibling artifact (`maturation-report.md`).

## Ground check (before)

- **Main branch:** `{main_branch}` @ `{baseline_main_head}`.
- **Backlog complete:** {sprint-status confirms all epics/lanes done} — *closeout never runs on an unfinished backlog.*
- **Gate green before closeout:** {typecheck ✓ · lint ✓ · test ✓ (N tests)} · tree clean ✓ — *you cannot validate a build that started red.*

## (A) Final E2E smoke campaign — the PROJECT validation gate

> By **user journey** (cross-epic), each delegated to **S5** and trusted without re-driving. These catch
> feature-interaction regressions no isolated wave could see. Expected figures were **re-derived from the
> source of truth**, not copied from a lane.

| Journey | Epics crossed | S5 verdict | Deltas (severity) | Figures (label = value ✓/✗) | Evidence |
| ------- | ------------- | :--------: | ----------------- | --------------------------- | -------- |
| {journey 1} | {e.g. 3 → 5 → 7} | {PASS/FAIL/BLOCKED} | {none / list} | {km = 127,20 € ✓, …} | {S5 evidencePaths} |
| {journey 2} | {…} | {…} | {…} | {…} | {…} |

- **renderDeferred at closeout:** {empty ✓ (every epic composed) / list — a non-empty list is a real gap, routed to triage}.
- **Runtime brought up once:** {emulator/browser + dev server — owner-launched; S5 attached as a consumer}.

### Triage of the campaign's work list

| Finding (journey · delta/figure/crash) | Route | Resolution |
| -------------------------------------- | ----- | ---------- |
| {trivial + targeted} | hotfix (fresh agent) + re-smoke | {commit · re-smoke verdict PASS} |
| {non-trivial / cross-cutting} | → consolidation (B) | {handled in §B item X} |
| {non-blocking} | → `deferred-work` / post-v1 | {logged with reason} |

## (B) De-parallelized consolidation wave

> **NOT** S3's parallel pipeline — consolidation is stitching/anti-isolation; this is the one deliberate
> de-parallelization. **Sequential**, lightened pipeline (careful dev + adversarial review + re-gate per
> item). No integration-collision phase exists here (that is the reward for going serial).

**Work list assembled from:** incurred debt (cross-lane duplications · seams/policies/FK blind-spots ·
worthwhile `deferred-work`) + planned debt (single-source consolidations that needed all consumers first)
+ the §A spillover.

| Item | Kind | Sites unified | Parity/regression test added | Review (max) | Commit | Re-gate |
| ---- | ---- | ------------- | ---------------------------- | ------------ | ------ | :-----: |
| {e.g. assembleFraisReelsInput} | {incurred-dup / planned / spillover} | {the N stores/lanes unified} | {permanent parity test path} | {PASS / patched} | {sha} | {✓ · test count {before}→{after}} |
| {…} | {…} | {…} | {…} | {…} | {…} | {…} |

- **No consumer result changed:** {proven by the parity tests — the unification is behavior-preserving}.
- **Re-smoke after consolidation:** {journeys whose screens B touched → re-run via S5 → verdict}.
- **Thin-orchestrator discipline:** {each refactor delegated to a fresh agent; orchestrator stayed the single writer to main}.

## Human validation

- **Triage signed off** (which hotfixed / consolidated / deferred): {name / pending}.
- **Consolidation validated** (main green, debt genuinely paid down, no consumer result changed): {name / pending}.
- **Maturation report exploited in the Builder** (reinjected, filtered by `fix-target = Sx`): {name / pending}.

## Hand-off

- **Maturation report:** `{output_folder}/pwo/maturation-report.md` (the durable, reinjectable deliverable).
- **Next:** `bmad-retrospective` — the product/process retro (a reused BMad skill, not reinvented here).
