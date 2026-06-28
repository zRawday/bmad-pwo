# Closeout Protocol — the orchestrator's recipe for the end-of-backlog bookend

The per-phase recipe SKILL.md routes here for: **§A** the final E2E smoke campaign (journey
decomposition + the S5 call shape + the triage protocol), **§B** the de-parallelized consolidation
wave (the work-list sources + the lightened sequential pipeline), and **§C** the maturation report
(the one-entry-per-problem schema + the aggregate-by-pointing rule). Read the section for the phase
you run. This file stands alone — it does not assume SKILL.md is still in context.

`REPO` = the absolute main-repo path · `MAIN` = `main_branch` · `OUT` = `{output_folder}/pwo`. The
two records you keep live at `OUT/closeout-record.md` (from `assets/closeout-record-template.md`) and
`OUT/maturation-report.md` (from `assets/maturation-report-template.md`).

**The standing discipline across all three phases** (do not relax it because this is "the end"): you
are a **thin orchestrator** — delegate every bulky thing (driving the emulator, a refactor, reading a
big diff) to a fresh agent and bring back a StructuredOutput, never a transcript; **verify from real
git / real output**, never the self-report; you are the **only writer** to `MAIN` and to the state
file. Closeout is where it is most tempting to "just do it inline" — resist; the orchestrator context
is a budget here too.

---

## §A — Final E2E smoke campaign (by user journey, each delegated to S5)

This is **the project validation gate**, and it is *not* the per-wave smoke repeated. The per-wave
smokes (S3) each proved *one increment* coherent against its mockup. The final campaign proves *all
the increments compose a coherent whole* — it walks **cross-epic user journeys end-to-end** to catch
the **feature-interaction regressions** no isolated wave could see (a value that flows correctly
through epic 5 but is mis-displayed once epic 7 composes it; a delete path that became FK-unsafe only
once a later epic added the first referencing rows). A green CI proves none of this — it renders no
pixels and enforces no FKs.

### 1. Decompose the app into cross-epic journeys

A *journey* is the whole flow a real user walks, **crossing epic boundaries** — not a single screen.
Derive them from the PRD's user flows + the whole-app mockups (P2) + the playbook's per-wave smoke
criteria (S1), then **compose** the per-wave fragments into end-to-end arcs. Aim for a small set
(≈3–6) that together exercise every cross-epic seam at least once. For each journey assemble a
**smoke criterion** — the exact contract you hand S5:

- **journey** — the ordered steps (navigate here, create X, add Y, expect screen Z with element W),
  spanning the epics it touches.
- **reference mockup(s)** — the P2 screens each asserted screen must match.
- **expected live figures** — the numbers that must render **exactly** (currency, sign, decimals),
  each **re-derived yourself from the source of truth** (the seeds / barèmes / formulas), never copied
  from a lane's self-report. These figures are the net that catches a wrong value a green test-double
  let through.

### 2. Bring up the runtime once; S5 is a consumer

S5 is a **consumer of running infra, never an owner** — it connects to an already-up
emulator/browser/device and the dev server, and must not boot a second one. So **before** the campaign,
ensure the `{smoke_harness}` runtime is up (you or the human launches the emulator + `npx expo start
--go` / the web server once). Each S5 call then attaches to it. If the runtime genuinely cannot come
up, S5 returns `BLOCKED` — that is the owner's to fix, not a FAIL.

### 3. Delegate each journey to S5 and trust its verdict

Call **`pwo-ui-smoke` (S5)** once per journey, passing it the smoke criterion above + the runtime
state. **Do not re-drive the app and do not re-read its screenshots** — S5 owns the capture-and-compare
contract and returns a compact StructuredOutput verdict; you consume it. The fields you read back (S5's
schema — consume it, don't re-declare it):

- `verdict` — `PASS | FAIL | BLOCKED` (PASS iff zero crashes, zero `blocking`/`major` deltas, every
  figure `match:true`).
- `deltas[]` — each `{ screen, expected, observed, severity }`, severity `cosmetic | major | blocking`.
- `figures[]` — each `{ label, expected, observed, match }`; a figure S5 couldn't locate on a real
  frame is `match:false`, not an optimistic guess.
- `crashes[]`, `renderDeferred[]` (a journey step whose nav entry doesn't exist — at closeout this
  should be **empty**; if it isn't, an epic never got composed → a real gap, route it to triage), and
  `evidencePaths[]` (screenshot files — record the paths, don't inline them).

Record each journey's verdict + `evidencePaths` in the closeout record. A campaign where every journey
PASSES is the project validation gate cleared.

### 4. Triage the produced work list

Each non-PASS verdict produces work. Triage every delta/crash/figure-mismatch **mechanically**:

| Finding | Route |
| ------- | ----- |
| **Trivial + targeted** (a single-screen cosmetic/major delta or a one-line figure fix, self-contained) | **Delegate a hotfix to a fresh agent** (paste-ready prompt: the delta, the file, the expected, "report a StructuredOutput") → **re-smoke that journey** via S5 to confirm. |
| **Non-trivial OR cross-cutting** (a regression spanning epics, a shared-helper bug, an FK/delete-path issue, anything touching code several lanes share) | **Into the consolidation wave (§B)** — it is exactly the holistic, sequential work B exists for. Do **not** hotfix it in isolation. |
| **Non-blocking** (a known limitation, a future-year/data gap, a cosmetic nicety) | **`deferred-work` / post-v1** — log it with the reason; it is not a closeout blocker. |

Re-smoke after every hotfix and after §B touches a journey's screens — a consolidation refactor can
regress UI, so close the loop before declaring the gate cleared.

---

## §B — De-parallelized consolidation wave (sequential, lightened pipeline)

**The headline invariant, restated because it is the whole point of this phase:** this wave **does
NOT use S3's parallel pipeline.** Consolidation is **stitching** — reuniting what isolation dispersed
— so it is *anti-isolation* by nature; running it as parallel lanes would re-create the very file
collisions it exists to resolve (the lanes would all be editing the shared homes you are trying to
unify). This is **the one deliberate de-parallelization in the whole method**: **reduced width /
sequential**, a **lightened pipeline** (a careful dev + an adversarial review + a re-gate per item),
**not** create→gate→dev→review→parallel-integration. We parallelize to *build* fast; we sequentialize
to *consolidate* cleanly.

### 1. Assemble the work list (two debts + the §A spillover)

Pull from the wave run-records (S3's end-of-wave critic flags), the Field Notes, `deferred-work`, and
the triage spillover from §A:

- **(1) Incurred debt — what isolation *created*** (subi):
  - **Cross-lane duplications.** N lanes that independently re-implemented one formula/predicate
    because no shared home existed yet (the critic flagged these and may have left a *temporary*
    on-master cross-check proving they agree). The fix: **extract one shared neutral helper** + add a
    **permanent parity test** asserting the consumers agree, then delete the duplicates. *(The
    canonical example: a cross-store `assembleFraisReelsInput` that two/three stores each assembled
    independently — prove byte-parity, then single-source it.)*
  - **Seams / policies / FK blind-spots to revisit holistically.** A forward-seam that moved; a
    mandatory policy a wave mocked away; a non-cascading-FK delete path that only the on-device smoke
    (or §A) could expose. These need a *whole-picture* fix, not a per-lane patch.
  - **`deferred-work` items worth doing now** (a mapper validation, a UX affordance the reviews parked).
- **(2) Planned debt — what *required all consumers to exist first*** (planifié): single-source
  consolidations that were **structurally deferred, not neglected** — you genuinely *could not* extract
  the shared helper until every consumer had shipped. Now that they all exist, do it.
- **(3) The §A spillover** — every "non-trivial / cross-cutting" finding the smoke triage routed here.

### 2. Work it sequentially under the lightened pipeline

One item at a time (or very low width on genuinely disjoint items — but default to sequential). Per item:

1. **Careful dev — delegate to a fresh agent** (paste-ready, self-contained prompt: the consolidation
   target, the duplicated sites to unify, the invariant to preserve, "add a permanent parity/regression
   test", "report a StructuredOutput"). Keep the orchestrator thin — you don't type the refactor. The
   dev works on `MAIN` directly or on a short-lived branch you ff-merge; there is **no worktree fan-out**.
2. **Adversarial review — top-level** (reuse `bmad-code-review`, effort `max`; it fans out, so it runs
   top-level, never nested — same Fact-1 constraint as S3). Same lenses: sweep don't spot-check; "is
   the policy tested, or mocked away?"; prove FK/delete order, not just zero-orphan. The review of a
   *consolidation* especially asks: **did unifying the duplicates change any consumer's result?** The
   parity test is the proof it didn't.
3. **Re-gate** (`typecheck → lint → test`) and verify the diff against **real git** (footprint is the
   consolidation's files; the test count grew additively — no loss). Commit small on `MAIN`
   (`refactor(consolidation): …` / `fix: …`). Then take the next item.

Because it is serial and re-gated per item, there is no integration-collision phase to resolve — that
cost (the C1 union/hotspot hand-resolution) **does not exist here**, which is the reward for
de-parallelizing. Record each item (target · duplicated sites unified · parity test added · review
verdict · commit) in the closeout record.

### 3. Validate + re-smoke

The consolidation touched cross-cutting code by definition, so re-run §A for any journey whose screens
it affected, and bring the consolidation log to the human for the validation sign-off (main green, the
debt genuinely paid down, no consumer result changed).

---

## §C — Method+skills maturation report (aggregate by pointing)

This is the **durable, reinjectable deliverable** — the one that makes the suite self-improving. It
**absorbs the former standalone "method retro"** (there is no separate method-retro document — keep it
here, no proliferation) and is **reinjectable into the BMad Builder**, filtered by `fix-target`, to
harden the very skills that produced it.

### The aggregation rule: POINT, don't duplicate

The Field Notes (FN-1…FN-N in the project's parallel-build run log) are the **dated evidence base** —
concrete, per-wave, already written. The report does **not** re-narrate them. It **points** at them
(by FN id, commit sha, deferred-work item, gate amendment, review escalation) and distills the recurring
signal across waves into **one entry per problem**. A friction that recurred every wave is **one** entry
with `recurrence: every wave (FN-2/4/5/7)`, not four entries. Dedup ruthlessly — the value is the
distilled, deduplicated, *actionable* view, not a transcript.

### Walk these sources

- the **Field Notes** (the run log) — the primary evidence base;
- the **wave run-records** (S3) — their end-of-wave critic flags, integration manual-resolutions, and
  code-review escalations;
- **`deferred-work`** — the parked items (which §B paid down, which remain);
- the **integration patches** + **gate amendments** (the run-records carry both) — what the orchestrator
  had to fix at integration or override at the gate;
- the **§A campaign + §B consolidation** you just ran — their findings are themselves maturation input.

### One entry per problem — the schema

Each entry (a row in the report table, or a short block):

| Field | What it holds |
| ----- | ------------- |
| **trace** | pointers to the evidence — `FN-x`, commit sha(s), `deferred-work` item, gate amendment id, review escalation. **Never the full story** — the pointer. |
| **recurrence** | how often / where it bit (`once @ W4` · `every wave (FN-2/4/5/7)` · `2 lanes W3`). |
| **severity** | impact if unaddressed (`silent corruption risk` · `money-affecting` · `friction/cost` · `cosmetic`). |
| **root cause** | the underlying mechanism, one line (not the symptom). |
| **category** | `intrinsic-parallelization` (a cost the parallelism itself creates — union corruption, cross-lane duplication, the FK-blind double, forward-seam staleness) · `stack-specific` (tied to the project's stack — a dep's moved export surface, the junction `node_modules`, a stale tool cache) · `harness` (tied to the orchestration harness — no-nesting, emulate-vs-invoke, the autocommit). |
| **status** | `lever N` (which retro lever already addresses it — 1=split test-ids, 2=FK-aware double/delete-path audit, 3=pre-provision deps, 4=gate fact-extraction, 5=end-of-wave critic, 6=failure-probe, 7=applicability gate, 8=coherence-sized waves, 9=thin orchestrator, 10=closeout) · or `open` (no lever yet). |
| **fix-target** | where the fix lands — `method doc §X` · **`skill Sx`** · `Phase 0` · `tooling` · `P2`. This is the routing key. |
| **action** | the concrete next step (one line). |

### Reinjectable into the Builder (the self-improving loop)

The report's payoff: **filter by `fix-target`**. Every entry with `fix-target = skill Sx` is a
**ready-made Edit brief** for that skill — the human (or you) reinjects those entries into
`bmad-workflow-builder` (Edit intent on skill `Sx`) to harden it; `fix-target = Phase 0` entries
sharpen S2's guard list; `fix-target = method doc §X` / `P2` entries mature the method/pre-conditions;
`fix-target = tooling` entries are infra work. Each project's closeout thus **hardens the skills that
produced it** — turn the reactive Field Notes into the next iteration's proactive guard-rails. Close the
report with the absorbed method-retro headline (verdict + the one or two highest-leverage open items)
so the maturation view is self-contained.

---

## After the closeout

Bring the closeout record to the human (the §A triage sign-off + the §B consolidation validation), then
point them at **`bmad-retrospective`** for the **product/process** retro — that stays a **reused BMad
skill, not reinvented here** (your report is the *method+skills* maturation; the retrospective is the
*product+process* review; they are complementary, not the same artifact).
