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
overall: {integrated | triaged | blocked | pending-human}   # integrated only when all lanes ff-merged, main green, smoke passed, human signed off; triaged when the wave closed on human-accepted debt
---

# Wave Run-Record — {wave id}

> One wave of the playbook, run end-to-end with main kept green. Everything here is verified against
> **real git / real output**, not a self-report. The **handoff prompt** at the bottom is the relay to
> the next wave's fresh session; the **Field Note** (if any) is appended to the project's parallel-build
> run log. S4 reads the accumulated run-records at closeout.

## Ground check (before)

- **Main branch:** `{main_branch}` @ `{baseline_main_head}` — matches the pasted handoff's claimed head: {yes / reconciled: …}.
- **Gate green before the wave:** {typecheck ✓ · lint ✓ · test ✓ (N tests)} — *you cannot prove you kept main green if it started red.*
- **Phase 0 receipt precondition:** `guard_proven_red: true` · `union_policy_proven: {true|n-a}` · `overall: ready` ✓ (from `{output_folder}/pwo/phase-0-receipt.md`).
- **Pre-wave `status --porcelain` baseline:** {captured → passed to every lane's leak-guard as the known pre-existing list}.

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
- **Spec-commit audit:** {`show --stat <specCommit>` = the one spec file ✓ per lane · no lane commit touches the state file (`log <MAIN>..HEAD -- <state-file>` empty) ✓}.
- **Test reality:** {each standard/critical lane's diff adds real tests · or lane {key} flagged to the reviewer (zero test files)}.
- **Actual disjointness:** {pairwise ∩ over real diff file lists = empty / every ∩ = declared hotspot · or the undeclared shared file + how it was resolved before integration}.
- **Hotspots:** {one new migration at its pre-allocated version (number verified against the playbook's allocation table); append-only registry blocks — verified}.
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
- **Re-gate after every rebase (incl. "clean"):** {✓ — N test runs}. Test count **{before} → {after}** additively (no loss). {Seam contract test un-skipped → skipped→green ✓ · n-a}.
- **Delta re-review:** {each manual hotspot resolution + each orchestrator-authored code commit → targeted top-level review (max) → verdict · or "none — no manual resolutions, no orchestrator code commits"}.
- **Junction-safe teardown:** link removed before `worktree remove`; main `node_modules` count **{n} → {n}** UNCHANGED ✓.
- **Orchestrator infra commits:** {state-file update (bmm vocabulary: story keys → done, epic-X transitions, last_updated bumped) · dep install + shim-drop + verified export surface · deferred-work ledger fold (DW-{wave}-{n}, …) · cleanup · a one-time seam fix (delta re-reviewed)} — each `chore(orchestration)`.

## End-of-wave critic (proactive)

- **Duplication-scout (delegated):** {ran over `{baseline}..{final}` → candidate pairs returned: {list / none}}.
- **Cross-lane duplications:** {N lanes re-implementing X (or re-implementing what main already had) → proven equal by a temp on-master cross-check → parity test + shared helper flagged for S4 · or none}.
- **Mocked-away policies:** {found / none}.
- **Moved forward-seams:** {found / none}.
- **Cross-lane idiom divergence:** {sampled side-by-side → divergences + the canonical idiom named → fixed / routed to S4 · or none}.
- **Flagged for S4 (deferred-work ledger):** {DW-{wave}-{n} appended to `{output_folder}/pwo/deferred-work.md` · or none}.

## End-of-wave smoke

- **Gate(s) by content:** {any engine lane → merged-result sweep · anything renders → S5 · mixed → both}.
- **S5 verdict:** `{PASS | FAIL | BLOCKED}` — journey *{played}*; deltas {none / list with severity}; figures {label = value ✓, …}; crashes {none}; renderDeferred {none / list}. **Envelope valid:** {harness ≠ none ✓ · uiBlindSpotCovered ✓ · evidence present ✓ · all asserted screens visited ✓}. **Evidence:** {S5 evidencePaths}. *(Trusted without re-driving; an invalid envelope = BLOCKED.)*
- **Merged-result sweep:** {jest green (N) + sweep over {range} by {runner/agent} → {pointsExecuted} points, mismatches {none/list} — outputTail captured: {excerpt/path}}.

## Smoke triage (only if the smoke was not a clean PASS)

| Finding (delta / figure / crash / blocked) | Route (hotfix / revert lane / next-wave lane / accepted debt DW-id) | Resolution (commit · review · re-gate) | Re-smoke |
| ------------------------------------------ | ------------------------------------------------------------------ | -------------------------------------- | -------- |
| {…} | {…} | {…} | {PASS / …} |

> Handoff emitted only after a re-smoke PASS or an explicit human acceptance (the accepted debt rides the handoff).

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

## Next-step handoff prompt (paste-ready, command-first — the human continues in a NEW session)

> **First token = the next command**, so pasting the block into a fresh session launches the skill
> directly. Use the **`/pwo-run-wave`** block when more waves remain; use the **`/pwo-closeout`** block
> below instead when this was the **LAST** wave (all lanes integrated, backlog complete).

```
/pwo-run-wave You are the PWO orchestrator (pwo-run-wave / S3) for {project-name}. Run the NEXT wave —
Wave {m} — end-to-end keeping `{main_branch}` green, in THIS fresh session.

START STATE (verified real git):
- `{main_branch}` @ {final_main_head} — gate green (typecheck ✓ · lint ✓ · test ✓, {N} tests).
- Phase 0 receipt: guard_proven_red: true / union_policy_proven: {true|n-a} / overall: ready.
- Backlog state: {which epics/lanes are done}; this wave's parents are all merged.

THIS WAVE — Wave {m} ({type: foundation | screen | mixed}, {k} lanes):
- Lanes + their ⚙ anti-collision constraints (VERBATIM, from the playbook's cards):
  - {key} ({tier}): footprint {…} · ⚙ "{constraint verbatim}"
  - …
- End-of-wave gate(s) — by content, a mixed wave carries BOTH: {any engine lane → jest + sweep of
  the MERGED result over {range} (runner: {…}) · anything renders → S5 smoke criterion:
  journey {…} · reference mockup(s) {paths} · expected live figures {label = value, …}}.
- Patterns digest (inject VERBATIM into every WF1/WF2 prompt; the card ⚙ constraint and any gate
  amendment OVERRIDE it where they conflict): {≤10 lines — established idioms / mistakes not to repeat}.
- Accepted debt carried forward: {DW-ids + one-liners · none}.

PIPELINE (do not skip a step): WF1 create (emulate) → GATE (read the real seam bodies, re-validate
FK ordering, cross-check load-bearing values, anti-mis-tier, bake any ⚙ gate amendment) → WF2 dev
(emulate; dev=xhigh, critical=max) → verify real git (§7) → code-review TOP-LEVEL (effort max;
sweep-don't-spot-check; "policy tested or mocked away?"; FK blind-spot; coherence) → serial
integration (rebase → resolve hotspots by hand → re-gate after EVERY rebase incl. "clean" +
un-skip any filled seam's contract test → ff-only → §7-verify merged → DELTA RE-REVIEW of every
manual resolution + orchestrator code commit → junction-safe teardown → orchestrator infra
commits incl. the deferred-work ledger) → end-of-wave critic (duplication-scout) → end-of-wave
smoke (by content; validate S5's envelope) → Field Note + next-wave handoff.

MUST NOT MISS: emulate create/dev (never invoke the Skill tool); you are the ONLY writer of the
state file; verify from real git, never the self-report; stay a thin orchestrator (delegate the
bulky, bring back a StructuredOutput); main green > speed > cost (route effort upward).

FIRST: read the playbook ({output_folder}/pwo/playbook.md) + the Phase 0 receipt
({output_folder}/pwo/phase-0-receipt.md) + the PREVIOUS wave's run-record ({path}) + the run log
({output_folder}/pwo/run-log.md), THEN propose the dispatch plan and get the human's GO before
launching WF1.
```

### …or, if this was the LAST wave (backlog complete) → close out instead of running another wave

```
/pwo-closeout All waves are integrated on `{main_branch}` @ {final_main_head} for {project-name}
(green; gate ✓). Close out the backlog: (A) run the final E2E smoke campaign by user journey — each
delegated to pwo-ui-smoke — and triage what it surfaces; (B) run the DE-parallelized consolidation
wave (sequential, lightened pipeline) to pay down the parallelization debt; (C) produce the maturation
report. Inputs: the wave run-records + the run log ({output_folder}/pwo/run-log.md, Field Notes), the
whole-app mockups (P2), the playbook ({output_folder}/pwo/playbook.md — incl. its Planned
consolidations section), and the deferred-work ledger ({output_folder}/pwo/deferred-work.md). Fresh
session.
```
