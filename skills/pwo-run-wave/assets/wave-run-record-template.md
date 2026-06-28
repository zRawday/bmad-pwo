---
title: Wave Run-Record
produced_by: pwo-run-wave (S3)
consumed_by: pwo-run-wave (S3) — the next wave · pwo-closeout (S4)
project: {project-name}
wave: {wave id / label}
ran: {timestamp}
baseline_main_head: {sha main was at before this wave}
final_main_head: {sha after the wave integrated}
smoke_verdict: {PASS | FAIL | BLOCKED | jest-only | n-a}
overall: {integrated | blocked | pending-human}   # integrated only when all lanes ff-merged, main green, smoke passed, human signed off
---

# Wave Run-Record — {wave id}

> One wave of the playbook, run end-to-end with main kept green. Everything here is verified against
> **real git / real output**, not a self-report. The **handoff prompt** at the bottom is the relay to
> the next wave's fresh session; the **Field Note** (if any) is appended to the project's parallel-build
> run log. S4 reads the accumulated run-records at closeout.

## Ground check (before)

- **Main branch:** `{main_branch}` @ `{baseline_main_head}`.
- **Gate green before the wave:** {typecheck ✓ · lint ✓ · test ✓ (N tests)} — *you cannot prove you kept main green if it started red.*
- **Phase 0 receipt precondition:** `guard_proven_red: true` · `overall: ready` ✓ (from `{output_folder}/pwo/phase-0-receipt.md`).

## Lanes (this wave)

| Lane | Tier | Spec commit | Gate amendment | Feat commit | Effort (dev) | Review | Footprint ✓ | Integrated |
| ---- | ---- | ----------- | -------------- | ----------- | :----------: | ------ | :---------: | :--------: |
| {key} | {trivial/standard/critical} | {sha} | {none / a one-line summary} | {sha} | {xhigh / max} | {PASS / patched / escalated} | {✓} | {✓ ff @ sha} |

## Gate amendments (authoritative overrides baked into the in-worktree specs)

> Each was a product/fiscal or seam-position decision taken at the gate (with the human where it was a
> product call); committed spec-only on the lane branch before WF2. The dev agents reported `amendmentHonored`.

- **{key}:** `⚙ ORCHESTRATOR GATE AMENDMENT` — {the authoritative instruction, e.g. "purge pending_distance BEFORE the étape delete"} · **human-decided:** {yes / n-a}.

## Verify with real git (§7)

- **Footprint:** {all lanes touched only their card's files / the exception + resolution}.
- **Hotspots:** {one new migration at its pre-allocated version; append-only registry blocks — verified}.
- **Producer→consumer:** {each lane consumed only on-main capabilities; no red post-rebase parent miss}.
- **Self-report vs real git:** {matched / the discrepancy + what it revealed} — re-checked via `worktree list` / `log` / `show --stat` / main `rev-parse`+`status`.

## Code-review (top-level, effort max)

- **Findings & patches:** {lane → finding → safe patch applied in-worktree + re-gated · or "0 defects"}.
- **Sweeps demanded:** {the numeric range swept per lane → result}.
- **Policy-tested-not-mocked:** {throttle/retry/auth verified with the stub OFF → result}.
- **FK blind-spot:** {delete-order proven where a lane added first FK-referencing rows · or n-a}.
- **Escalations:** {any decision-needed → how resolved}.

## Serial integration (DAG → ascending schema version)

- **Order:** {key, key, …}.
- **Manual hotspot resolutions:** {file → shape (silent test-ids splice / partial .test.ts close / glyph keep-both / store keep-HEAD) → fixed + re-gated}. *(Phase 0's test-id split should have made these rare.)*
- **Re-gate after every rebase (incl. "clean"):** {✓ — N test runs}. Test count **{before} → {after}** additively (no loss).
- **Junction-safe teardown:** link removed before `worktree remove`; main `node_modules` count **{n} → {n}** UNCHANGED ✓.
- **Orchestrator infra commits:** {state-file update · dep install + shim-drop + verified export surface · cleanup · a one-time seam fix} — each `chore(orchestration)`.

## End-of-wave critic (proactive)

- **Cross-lane duplications:** {N lanes re-implementing X → proven equal by a temp on-master cross-check → flagged a parity test + shared helper for S4 · or none}.
- **Mocked-away policies:** {found / none}.
- **Moved forward-seams:** {found / none}.

## End-of-wave smoke

- **Type:** {foundation → jest + sweep · screen → S5}.
- **S5 verdict (screen-waves):** `{PASS | FAIL | BLOCKED}` — journey *{played}*; deltas {none / list with severity}; figures {label = value ✓, …}; crashes {none}; renderDeferred {none / list}. **Evidence:** {S5 evidencePaths}. *(Trusted without re-driving.)*
- **Foundation verdict:** {jest green (N) + sweep over {range} → result}.

## Human validation

- **Integration validated on main (green):** {name / pending}.
- **Smoke verdict received:** {name / pending}.
- **Handoff approved:** {name / pending}.

---

## Field Note (append to the run log only if the wave taught something new)

```
FN-{n} · {date} · {project} Wave {id} ({k} lanes: {key — one-liner}, …).
{The load-bearing learnings — what recurred, what a guard caught, what a new case taught. Be concrete and
 dated; point at the real commit/output. This is the evidence base S4's maturation report aggregates.}
```

---

## Next-wave handoff prompt (paste-ready — the human starts the next wave in a NEW session with this)

```
You are the PWO orchestrator (pwo-run-wave / S3) for {project-name}. Run the NEXT wave —
Wave {m} — end-to-end keeping `{main_branch}` green, in THIS fresh session.

START STATE (verified real git):
- `{main_branch}` @ {final_main_head} — gate green (typecheck ✓ · lint ✓ · test ✓, {N} tests).
- Phase 0 receipt: guard_proven_red: true / overall: ready.
- Backlog state: {which epics/lanes are done}; this wave's parents are all merged.

THIS WAVE — Wave {m} ({type: foundation | screen}, {k} lanes):
- Lanes + their ⚙ anti-collision constraints (VERBATIM, from the playbook's cards):
  - {key} ({tier}): footprint {…} · ⚙ "{constraint verbatim}"
  - …
- End-of-wave gate: {foundation → jest + sweep over {range} · screen → S5 smoke criterion:
  journey {…} · reference mockup(s) {paths} · expected live figures {label = value, …}}.

PIPELINE (do not skip a step): WF1 create (emulate) → GATE (read the real seam bodies, re-validate
FK ordering, cross-check load-bearing values, anti-mis-tier, bake any ⚙ gate amendment) → WF2 dev
(emulate; dev=xhigh, critical=max) → verify real git (§7) → code-review TOP-LEVEL (effort max;
sweep-don't-spot-check; "policy tested or mocked away?"; FK blind-spot) → serial integration
(rebase → resolve hotspots by hand → re-gate after EVERY rebase incl. "clean" → ff-only →
junction-safe teardown → orchestrator infra commits) → end-of-wave critic → end-of-wave smoke →
Field Note + next-wave handoff.

MUST NOT MISS: emulate create/dev (never invoke the Skill tool); you are the ONLY writer of the
state file; verify from real git, never the self-report; stay a thin orchestrator (delegate the
bulky, bring back a StructuredOutput); main green > speed > cost (route effort upward).

FIRST: read the method (`bmad-parallel-implementation-method.md`) + the playbook
(`{output_folder}/pwo/playbook.md`) + your memory, THEN propose the dispatch plan and get the
human's GO before launching WF1.
```
