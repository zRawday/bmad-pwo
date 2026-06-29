---
name: pwo-closeout
description: "STEP 6 of 6 — the final step. Close out a finished parallel backlog — run the final E2E smoke campaign by user-journey (each via S5), pay down the parallelization debt in a DE-parallelized consolidation wave, and produce the method+skills maturation report. Use when the user says \"run closeout\", \"pwo closeout\", \"close out the build\", \"final smokes + consolidation\", \"PWO S4\", or once all waves are integrated."
---

# pwo-closeout

Validate the **whole project** end-to-end and pay down the debt the parallelization incurred — the
**bookend symmetric to Phase 0** (`pwo-build-phase0`, S2 prepared the ground *before* the waves; you
validate and repair *after* them). Run **once, at end-of-backlog**, when every wave is integrated on
main. You do three things, in order: **(A)** run the **final E2E smoke campaign** — by user journey,
cross-epic, each delegated to **`pwo-ui-smoke` (S5)** — and triage what it surfaces; **(B)** run the
**de-parallelized consolidation wave** that pays down the incurred + planned debt; **(C)** produce the
**maturation report** that feeds the suite's own improvement. Then hand off to **`bmad-retrospective`**
(the product/process retro — a reused BMad skill, not reinvented). You are an **orchestrator**:
delegate everything bulky (the E2E smokes, the consolidation refactors) to fresh agents, keep only the
targeted load-bearing judgement.

**The non-negotiable (the most counter-intuitive thing here):**

**The consolidation wave does NOT use S3's parallel pipeline.** Consolidation is **stitching** — the
*anti-isolation* work of reuniting what the waves dispersed. Parallelizing it would re-create the
exact collisions it exists to resolve. This is **the one moment the method de-parallelizes on
purpose**: reduced width / sequential, a **lightened pipeline** — careful dev + adversarial review,
**not** create→gate→dev→review→parallel-integration. We parallelize to *build* fast; we sequentialize
to *consolidate* cleanly. Encode this in every consolidation-agent prompt.

And the standing priorities, which never bend:

1. **The final smokes are the PROJECT validation gate** — distinct from the per-wave smokes (those
   proved *each increment* coherent; these prove *all increments compose a coherent whole*),
   organized by **user journey** to catch the **feature-interaction regressions** no isolated wave
   could see. Each journey is **delegated to S5** — trust its StructuredOutput verdict; **never
   re-drive the app or re-read its screenshots**.
2. **The maturation report aggregates by POINTING, never duplicating** — it points to the Field
   Notes / `deferred-work` / integration patches / gate amendments / review escalations; one entry
   per problem. It **absorbs the former standalone "method retro"** (no proliferation), and is
   **reinjectable into the Builder** (filter `fix-target = Sx`).
3. **Inherited cross-cutting invariants** — verify from the source of truth, never the self-report
   (#1); stay a **thin orchestrator**, delegate the bulky E2E smokes + consolidation refactors to
   fresh agents and bring back a StructuredOutput, not a transcript (#2); you are the **only writer**
   of the state file / main branch (#3); **emulation stays the default** even in the lightened
   pipeline (#5); main green > speed > cost (#6).

## Resolution rules

- Bare paths and `{skill-root}` (e.g. `references/closeout-protocol.md`) resolve from this skill's installed directory.
- `{project-root}` → the project working directory.
- `pwo-closeout` → the skill directory's basename.

## On Activation

Human-triggered, single-pass at end-of-backlog. **No runtime memlog, no resume store**: durable state
**is git** (the commits the consolidation lands) plus the **two artifacts** you write as you go — to
resume an interrupted closeout, re-read them and reconcile against real git, never a remembered state
(#1). Talk to the human in `{communication_language}`.

1. **Config.** Read `{project-root}/_bmad/config.yaml` (and `.user.yaml` if present), defaults for
   anything missing. Resolve `{project-name}`, `{output_folder}` (else `{project-root}/_bmad-output`),
   `{communication_language}`, `main_branch` (else `git rev-parse --abbrev-ref HEAD`), `smoke_harness`,
   `worktree_workspace`.
2. **Inputs & preconditions** (each is a hard stop unless noted):
   - **all waves integrated** — confirm from `sprint-status` (the state file the orchestrator owns)
     that the backlog is complete, and confirm the gate chain (`typecheck → lint → test`) is **green
     and the tree clean** from real git. Record the baseline HEAD. Closeout never runs on an
     unfinished or red backlog. Absent / red → **stop & escalate**.
   - **the wave run-records** — the accumulated `wave-run-record` outputs from S3 (each wave's
     verified outcome + its end-of-wave critic flags) and the project's **parallel-build run log**
     (the dated **Field Notes**, FN-1…FN-N). These are the raw material for the maturation report and
     the source of the *planned* + *flagged* debt. Absent → the report degrades to what you can verify
     from git + `deferred-work`; **note the gap**, don't stop.
   - **the smoke material** — the **whole-app mockups (P2)** and the **playbook** (S1) journeys /
     expected figures, to build the E2E campaign's smoke criteria. Absent → **stop & escalate** (a
     smoke with no mockup is a no-crash check, not the validation gate).
   - **`deferred-work`** — the parked items each wave's review filed (the consolidation's work-list seed).

## The closeout — run A → B → C in order, keep the two records

The concrete recipe for each phase lives in **`references/closeout-protocol.md`** — read the section
for the phase you run. Record the campaign + triage + consolidation log into the artifact from
`assets/closeout-record-template.md` (the bookend receipt, symmetric to S2's), and the distilled
findings into `assets/maturation-report-template.md` (the durable, reinjectable deliverable).

### (A) Final E2E smoke campaign — by user journey, each delegated to S5

Decompose the app into **cross-epic user journeys** (not per-screen — the whole flow a real user
walks). Delegate each to **S5** with its journey + reference mockup(s) + expected live figures, and
**trust its verdict** (`PASS | FAIL | BLOCKED`, with `deltas` + per-figure `match`). Then **triage**
the produced work list: **trivial-targeted** → delegate a hotfix to a fresh agent + re-smoke;
**non-trivial / cross-cutting** → into the consolidation wave (B); **non-blocking** → `deferred-work`
/ post-v1. (Recipe + the S5 call shape + the triage table: `references/closeout-protocol.md` §A.)

### (B) De-parallelized consolidation wave — sequential, lightened pipeline

Assemble the work list — **(1) incurred debt** (duplications isolation created, e.g. a cross-store
`assembleFraisReelsInput` three lanes re-implemented; `deferred-work` items worth doing; seams /
policies / FK blind-spots to revisit holistically) + **(2) planned debt** (single-source consolidations
that *required all consumers to exist first* — structurally deferred, not neglected) + the triage
spillover from (A). Then work it **sequentially** under the **lightened pipeline** (careful dev +
adversarial review, re-gate after each; **never** the parallel S3 machinery). Delegate each non-trivial
refactor to a fresh agent; you stay the single writer to main. (Recipe: `references/closeout-protocol.md` §B.)

### (C) Method+skills maturation report — aggregate by pointing

Walk the Field Notes + `deferred-work` + the integration patches + gate amendments + review escalations
and distill them into **one entry per problem**: trace · recurrence · severity · root cause · **category**
(`intrinsic-parallelization | stack-specific | harness`) · **status** (`lever N | open`) · **fix-target**
(`method doc §X | skill Sx | Phase 0 | tooling | P2`) · action. **Point** at each source (the FN id, the
commit, the deferred item) rather than re-narrating it. This artifact **absorbs the ex-"method retro"** and
is **reinjectable into the Builder** filtered by `fix-target`. (Schema + the aggregate-don't-duplicate
rule: `references/closeout-protocol.md` §C.)

Then bring the closeout record to the human (the consolidation validation + the triage sign-off), and
point them at **`bmad-retrospective`** for the product/process retro.

## Human intervention

**Patch triage** (after the final smokes — which trivial-hotfix, which to the consolidation, which to
defer), **validating the consolidation** (it touched cross-cutting code — confirm main is green and the
debt genuinely paid down), and **exploiting the maturation report in the Builder** (the human reinjects
it, filtered by `fix-target`). Headless → mark each `pending-human` (with the real-git evidence) and
emit the JSON tail.

## Hand off to the next step (the retro)

Closeout is the **last PWO step**, but the pipeline continues into BMad's reused product/process retro,
so close the loop the same way every other step does: hand the user a paste-ready prompt for
**`bmad-retrospective`**, run in a **fresh session**. Once the consolidation is validated and both
records are written, emit — in **English** — a single fenced code block whose **first token is the next
command** followed by a self-contained prompt, then tell the user: **"Copy this into a NEW Claude Code
session (fresh context) to run it."** Resolve every `{token}` to its real value (the final main `{sha}`,
the two artifact paths, the project name). Skip in headless mode (the JSON tail carries
`closeoutRecord` + `maturationReport`).

```
/bmad-retrospective Run the product/process retrospective for {project-name}. The PWO build is closed
out: main is green @ {sha}, the final E2E smokes passed, the de-parallelized consolidation wave is
merged, and the maturation report is at {output_folder}/pwo/maturation-report.md (closeout record:
{output_folder}/pwo/closeout-record.md). Use the wave run-records + Field Notes + maturation report as
input on what the parallel build delivered and what debt was paid down vs. deferred. Fresh session.
```

## Headless output

When invoked headless, run A→B→C, write both records, mark the human checkpoints `pending-human`, and
return JSON only —
`{ "status": "complete" | "blocked", "mainHead": "<sha>", "journeysSmoked": ["<journey>", …], "smokeVerdicts": { "<journey>": "PASS" | "FAIL" | "BLOCKED" }, "consolidationCommits": ["<sha>", …], "debtPaid": ["<item>", …], "maturationEntries": <n>, "closeoutRecord": "<path>", "maturationReport": "<path>", "pendingHuman": ["patch-triage", "consolidation-validation", "maturation-report-exploited"] }`
(`"blocked"` with a one-line `reason` if the backlog is not complete, if main was red before you
started, or if the mockups/playbook needed for the E2E campaign are absent — those preconditions are
never waived).
