---
title: Phase 0 Receipt
produced_by: pwo-build-phase0 (S2)
consumed_by: pwo-run-wave (S3)
project: {project-name}
built: {timestamp}
baseline_main_head: {sha main was at before Phase 0}
final_main_head: {sha after Phase 0 merged}
guard_proven_red: {true | false}
overall: {ready | blocked | pending-human}   # ready only when merged + green + guard proven RED + human signed off
---

# Phase 0 Receipt

> The safety infrastructure the playbook's `Phase 0 spec` required, implemented and **proven** on
> the main branch before Wave 1. S3 keys off this receipt — specifically `guard_proven_red` and the
> RED-proof block below. Everything here is verified against **real git / real output**, not a
> self-report.

## Ground check (before)

- **Main branch:** `{main_branch}` @ `{baseline_main_head}`.
- **Tree clean at start:** {yes / pre-existing untracked ignored: …}.
- **Gate green before Phase 0:** {typecheck ✓ · lint ✓ · test ✓ (N tests)} — *you cannot prove you
  kept main green if it started red.*

## Items built

| Item | Status | Artifact / files | Commit | Gate |
| ---- | :----: | ---------------- | ------ | :--: |
| Migration-set guard | {done / n-a} | {test path} | {sha} | {✓} |
| Cross-wave seams | {done / n-a} | {port/stub paths} | {sha} | {✓} |
| `.gitattributes` policy | {done / n-a} | `.gitattributes` | {sha} | {✓} |
| Native deps pre-provisioned | {done / n-a} | {package.json + lock} | {sha} | {✓} |
| Test-ids split per screen | {done / n-a} | `test-ids/*` + barrel | {sha} | {✓} |
| Worktree workspace + ownership | {done / n-a} | {workspace path} | {sha} | {—} |

> `n-a` items: {which, and why the project doesn't need them — e.g. "no SQLite migrations; no native deps"}.

## RED proof (the load-bearing evidence)

> S3 reads this as proof the migration-set guard is real, not assumed (blind spot A3).

- **Fault injected:** {duplicate version vN / out-of-order / DB_VERSION≠count / duplicate CREATE TABLE — be specific}.
- **Isolation:** {throwaway worktree `<path>` (torn down junction-safe) / working-tree-only edit reverted}.
- **Real failing output (excerpt):**
  ```
  {the actual assertion message + non-zero exit — pasted, not paraphrased}
  ```
- **Reverted & green-after:** {migration set restored ✓ · full gate green ✓ · `git status` clean ✓}.

## Seams hoisted (position is the value)

| Port | Consumer | Position / ordering constraint baked in |
| ---- | -------- | --------------------------------------- |
| {name} | {wave/story} | {esp. FK ordering: purge/delete BEFORE the referenced-row delete — and why} |

## Native deps — verified surface

| Dep@version | Peer/transitive added | Real export surface (verified) |
| ----------- | --------------------- | ------------------------------ |
| {dep@v} | {peers pulled / a missing peer installed separately} | {actual import path + exported names — e.g. fs moved to `/legacy`} |

## Test-ids split

- **Per-screen files:** {`test-ids/<screen>.ts`, …}
- **Barrel:** `test-ids/index.ts` re-exports each — existing call sites unchanged; lanes append only their own screen file (zero union, zero collision — lever 1 / cost C1 closed).

## Ownership policy (the waves obey this)

- **Lanes NEVER write the coordination/state file** ({sprint/status path}); the **orchestrator owns it 100%**, updating main at integration to reflect merged reality.

## Human validation (gates the waves)

- **Phase 0 merged on main, green, guard proven RED:** {name / pending} — verified against real git
  (final HEAD `{final_main_head}`, the gate output, the RED excerpt above), not a summary.
