---
name: pwo-run-wave
description: "STEP 5 of 6 — repeat once per wave, each in a fresh session. Execute ONE parallel wave end-to-end keeping main green — emulate create/dev, gate, top-level review, serial integration, smoke, handoff. Use when the user pastes the previous wave's handoff prompt, says \"run the wave\", \"run wave N\", \"pwo run wave\", or \"PWO S3\"."
---

# pwo-run-wave

Execute **one wave** of the parallelization playbook end-to-end while the **main branch stays
green** — the central operational skill, invoked **once per wave in a fresh session** (§13 bounds
the orchestrator's context budget). You consume the **playbook** (`pwo-plan-waves`, S1) for this
wave's lane cards + end-of-wave gate, and key off the **Phase 0 receipt** (`pwo-build-phase0`, S2)
— you do not start until its guard is proven RED. A wave is a fixed pipeline: **WF1 create
(emulate) → GATE → WF2 dev (emulate) → verify-real-git → code-review (top-level) → serial
integration → critic → smoke → Field Note + next-wave handoff.** You are an **orchestrator**:
delegate everything bulky, keep only the targeted load-bearing judgement.

**The two non-negotiables:**

1. **Emulate create/dev — never invoke the Skill tool** (harness Fact 2). The real
   `create-story` / `dev-story` load `{project-root}` from the **main repo**, so their writes (and
   `dev-story`'s `git add -A` autocommit) leak into the tree N lanes are racing. Each lane agent
   **authors the spec / implements directly to its absolute worktree path**, commits
   **footprint-only**, runs a **main-repo leak-guard**. The most counter-intuitive invariant and
   the easiest to fake — encode it concretely in every WF prompt.
2. **Code-review runs TOP-LEVEL only** (harness Fact 1: it fans out → cannot be nested in a
   workflow). Reuse `bmad-code-review` at effort **max**; **sweep, don't spot-check**; ask **"is
   the policy tested, or mocked away?"**; carry the **FK blind-spot** (the in-memory double
   enforces no FKs).

And the standing priorities, which never bend: **the orchestrator is the only writer of the
state file** (`sprint-status`) — lanes never touch it; **verify from the source of truth, never
the self-report** — re-check every lane with real git (self-reports matched real git 100% and the
re-check is still mandatory); **stay a thin orchestrator** — bring back a StructuredOutput, never
a transcript or a 500-line diff; **main green > speed > cost** — route effort *upward*, and route
the **model** deliberately: you choose model AND effort for every delegation (the model × effort
table in the reference); the session's top-tier model is reserved for **your own** judgement and
lands on a sub-agent only on explicit user request.

## Resolution rules

- Bare paths and `{skill-root}` (e.g. `references/wave-pipeline.md`) resolve from this skill's installed directory.
- `{project-root}` → the project working directory.
- `pwo-run-wave` → the skill directory's basename.

## On Activation

Human-triggered, **one fresh session per wave**, started by the human pasting the previous wave's
handoff prompt. **No runtime memlog, no resume store**: durable state **is git** (the commits you
land) plus the **wave run-record** you write as you go — to resume an interrupted wave, re-read the
run-record and reconcile it against real git, never a remembered state. Talk to the human in
`{communication_language}`.

1. **Config.** Read `{project-root}/_bmad/config.yaml` (and `.user.yaml` if present), defaults for
   anything missing. Resolve `{project-name}`, `{output_folder}` (else `{project-root}/_bmad-output`),
   `{communication_language}`, `main_branch` (else `git rev-parse --abbrev-ref HEAD`),
   `smoke_harness`, `worktree_workspace`, and **`implementation_artifacts`** (bmm's spec/state dir —
   fallback `_bmad-output/implementation`; the lane specs MUST land there or bmm's own tooling never
   finds them).
2. **Inputs & preconditions** (each is a hard stop):
   - **playbook** — `{output_folder}/pwo/playbook.md` (S1): this wave's lane cards (footprint ·
     hotspots · the ⚙ constraint *verbatim* · tier) and end-of-wave gate(s) (the merged-result
     sweep for any engine lane, the **smoke criterion** — journey + reference mockup(s) + expected
     live figures — when anything renders; a mixed wave carries both).
     Resolve **which wave** from the handoff prompt / the user / `sprint-status`. Absent → **stop & escalate**.
   - **Phase 0 receipt** — `{output_folder}/pwo/phase-0-receipt.md` (S2): require `guard_proven_red:
     true`, `union_policy_proven: true` (or `n-a`), and `overall: ready`. Else → **stop & escalate**
     (the waves key off a *proven* Phase 0).
   - **green main** — confirm the gate chain (`typecheck → lint → test`) is green and the tree clean
     from real git **before** you start (you can't prove you kept main green if it began red).
     Record the baseline HEAD — and **cross-check the handoff prompt's claimed `final_main_head`
     against it**: a mismatch means a stale handoff (or an unrecorded change); reconcile before
     proceeding, with the precedence **real git > run-records > sprint-status > the pasted prompt**.

## The pipeline — run it in order, keep the run-record

The steps below, in order; each feeds the next. The **recipe** for every step — the WF1/WF2 script
skeletons + StructuredOutput schemas + the leak-guard/footprint-only parade, the **model × effort table**,
the gate fact-extraction recipe, the §7 real-git checklist, the serial-integration + junction-safe
teardown order, the critic checklist — lives in **`references/wave-pipeline.md`**; read the section
for the step you run. Record each step into the artifact from `assets/wave-run-record-template.md`
(your only durable working state).

### WF1 create — parallel, EMULATE

One workflow agent per lane: cut a **durable** worktree + branch from the same main snapshot,
**survey main first** (grep for existing implementations → a "Consumes from main" list; read the
full body of every UPDATE file in the footprint → a preserved-behaviors Dev Notes block), then
**emulate create-story** to author the spec — from bmm's installed `create-story` template (wave
2+: a merged prior spec works as the format reference) — directly to its absolute worktree path
with the card's **⚙ constraint baked verbatim**, commit **spec-only**, run the **report-only**
leak-guard (a lane never writes, cleans, or deletes anything in the main repo — it reports and the
orchestrator triages). **Opus-class, effort high.**

### GATE — the fact-extraction pre-pass (you; load-bearing, irreplaceable)

The one step you cannot delegate. First **reconcile this wave's cards with merged reality** (the
playbook froze pre-code: a stale footprint/anchor is amended in the playbook like a gate
amendment; a wrong/missing edge defers the lane — or escalates to S1's scoped re-plan). Then
**read the real body of every seam/function the lane consumes — not the spec prose.** A
forward-seam comment left by an earlier wave encodes *that* wave's assumptions, so **re-validate
its position against this consumer's new constraints**, above all **FK ordering** (purge/child
delete *before* the referenced-row delete). Verify the ⚙ constraint is baked, the footprint and
ACs match, the spec's structure is real (Tasks mapped to ACs), the **"Consumes from main" list
matches the real exports** (a spec re-specifying an existing helper goes back), the
**preserved-behaviors block exists for every UPDATE file**, and **cross-check every load-bearing
value** (seeds/barèmes) against the source of truth. Apply the **anti-mis-tiering guard** (a
`critical` lane whose spec reads thin goes back). Where the gate settles a **product/fiscal
decision with the human**, bake a prominent `⚙ ORCHESTRATOR GATE AMENDMENT — OVERRIDES
CONFLICTING PROSE BELOW` block at the **top** of the in-worktree spec, fix the 1–2 contradicting
lines, commit spec-only before WF2.

### WF2 dev — parallel, EMULATE

One agent per lane **reuses** its worktree, junctions `node_modules` (**never install through the
junction**), **emulates dev-story** directly to absolute worktree paths — **red→green per AC**
(write the failing test first; implement nothing unmapped to a spec Task) — gates **in the
worktree** (never green by weakening an assertion or re-implementing a missing parent symbol:
that returns red with the reason — the DAG may be wrong, and that is the orchestrator's call),
commits **footprint-only**, runs the **report-only** leak-guard. **Route model and effort**: dev =
**opus·xhigh**, **opus·max on a `critical` lane** (the tier comes from S1's card; the top-tier
session model never runs a lane absent explicit user request). **Robustness:** bounded
retry on a red gate; `resumeFromRunId` if a lane dies, to protect the xhigh/max investment.

### Verify with real git (you)

Re-check **every** lane against real git (`worktree list`, `log`, `show --stat`, `diff --stat`,
main `rev-parse`/`status`), never the self-report: **footprint compliance**, the **spec-commit
audit** (`show --stat <specCommit>` = the one spec file; no lane commit touches the state file),
**test reality** (the diff adds real tests — a standard/critical lane adding zero test files goes
to the reviewer flagged), **actual disjointness** (recompute the pairwise intersection over the
lanes' real diff file lists — an undeclared shared file stops the integration order), **hotspot
discipline** (one new migration at its pre-allocated version — verify the *number* against the
playbook's allocation table; append-only registry blocks), **registry-merge integrity**,
**producer→consumer** sanity (the lane consumes only on-main capabilities).

### Code-review — TOP-LEVEL only

Run `bmad-code-review` top-level (Fact 1) at **opus·max**, one pass per lane, with the diff range
+ spec (incl. preserved-behaviors) + invariants, and its **write-backs neutralized** (the reviewer
writes nothing under the main repo; defer-findings return in the StructuredOutput; patches commit
on the lane branch; no lane integrates with an open finding). **Sweep the realistic input range**
for numeric/format code; verify any **mandatory side-effecting policy** (throttle/retry/auth) is
exercised with its stub **off**; carry the **FK blind-spot**; check **coherence** with the
project's conventions (the fourth lens). Apply safe patches **in the worktree + re-gate**; a
load-bearing patch is **opus·xhigh**.

### Serial integration (you; incompressible)

Per lane in **DAG-then-ascending-schema-version** order: sync main → **rebase** → **resolve any
registry/test-id/glyph hotspot BY HAND** (Phase 0's per-screen split should have killed most; the
fix is deterministic — keep every lane's block) → **re-gate after EVERY rebase, including a
"clean" one** (union corrupts `test-ids.ts` *silently*, even when git reports the rebase
conflict-free; only `tsc` catches it; a lane filling a hoisted seam also un-skips its contract
test here) → `merge --ff-only` → §7 verify → **delta re-review** (every by-hand hotspot
resolution and every orchestrator-authored code commit gets a targeted top-level review at max —
the merged content must not silently diverge from the reviewed content) → **junction-safe
teardown** (remove the LINK *before* `worktree remove`; verify the main `node_modules` count
unchanged). Then your **own** `chore(orchestration)` commits on main: the **state file** (you own
it 100% — write it in **bmm's status vocabulary only**, per the reference), any **lane-declared
dependency** install (verify the real export surface), the **deferred-work ledger** fold
(`{output_folder}/pwo/deferred-work.md`, `DW-` ids), cleanup.

### End-of-wave critic (you; internal step)

A proactive pass over the merged wave for what isolation hides — detection delegated (the
**duplication-scout**, sonnet·high, over the wave's merged diff: new exported symbols clustered
against the wave's other exports AND pre-existing main; duplicated fixtures; contradictory
seeds), verdict kept: **cross-lane duplications**, **mocked-away policies**, **forward-seams that moved**,
**cross-lane idiom divergence**. Fix trivial now (delegate any non-trivial change to a **fresh
agent** — stay thin); flag a cross-cutting one for the closeout consolidation (S4) as a `DW-`
ledger entry.

### End-of-wave smoke

The gate follows the wave's **content**: **anything renders** → delegate to **`pwo-ui-smoke`
(S5)** at sonnet·high: pass it exactly the wave's **smoke criterion** (journey + reference
mockup(s) + expected live figures) and **trust its StructuredOutput verdict** (PASS/FAIL/BLOCKED
+ deltas + figures) **without re-driving the app** — but **validate its envelope** (PASS counts
only with `harness ≠ none`, `uiBlindSpotCovered:true`, evidence present, every asserted screen
visited; else treat as BLOCKED). **Any engine lane** → **jest + the empirical sweep of the merged
result**, executed by a delegated agent (sonnet·high) with captured output — never narrated. A
mixed wave runs **both**.
Record the verdict; a FAIL/BLOCKED routes to **Smoke triage** (`references/wave-pipeline.md`
§ *Smoke triage*), not the handoff.

### Field Note + handoff (you; the relay)

If the wave taught something new, append a **dated Field Note** to the parallel-build run log at
**`{output_folder}/pwo/run-log.md`** (create it at wave 1 — S4 names this log its primary evidence
base; a Field Note in any other location is a Field Note S4 loses). Distill the **≤10-line
patterns digest** for the next wave's WF1/WF2 prompts (established idioms / mistakes not to
repeat; card constraints and amendments override it). Then write the **run-record** + a
**self-contained next-wave handoff prompt** from `assets/wave-run-record-template.md`: the current
**main SHA + green state**, the next wave's **lanes + their ⚙ constraints**, the pipeline, the
must-not-miss practices, the patterns digest, any **accepted debt** (`DW-` ids), "**read the
playbook + Phase 0 receipt + the previous run-record + the run log first**", "**propose the
dispatch plan and get the human's GO before WF1**". The human starts the next wave in a **new**
session by pasting it.

**Command-first, and branch on whether the backlog is done.** The handoff prompt's **first token must be
the next command** so a paste into a fresh session launches it directly (the template in
`assets/wave-run-record-template.md` now leads with it): use **`/pwo-run-wave`** when more waves remain,
or **`/pwo-closeout`** when this was the **last** wave (all lanes integrated, backlog complete) — in
which case the prompt instead points at the final E2E smokes + de-parallelized consolidation. Emit the
block in **English**, then tell the user: **"Copy this into a NEW Claude Code session (fresh context) to
run it."** A **FAIL** smoke routes to triage, not to the handoff.

## Human intervention

The **GATE** (load-bearing judgement, incl. product/fiscal gate amendments), **validating serial
integration**, **receiving the S5 smoke verdict**, the **handoff**. Headless → mark each
`pending-human` (with the real-git evidence) and emit the JSON tail.

## Headless output

When invoked headless, run the full pipeline, write the run-record, mark the human checkpoints
`pending-human`, and return JSON only —
`{ "status": "complete" | "blocked", "wave": "<id>", "mainHead": "<sha>", "lanesIntegrated": ["<key>", …], "gatesGreen": true | false, "smokeVerdict": "PASS" | "FAIL" | "BLOCKED" | "jest-only", "fieldNoteAppended": true | false, "runRecord": "<path>", "handoffPrompt": "<path>", "pendingHuman": ["gate", "integration-validation", "smoke-verdict", "handoff"] }`
(`"blocked"` with a one-line `reason` if the playbook or this wave's cards are absent, if the Phase 0
receipt is not `guard_proven_red: true` / `union_policy_proven: true|n-a` / `overall: ready`, or if
main was red before you started — those preconditions are never waived).
