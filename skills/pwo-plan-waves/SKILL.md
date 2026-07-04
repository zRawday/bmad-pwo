---
name: pwo-plan-waves
description: "STEP 3 of 6. Turn epics + architecture into a safe parallelization playbook (applicability gate, adversarial DAG, waves, Phase 0 spec, per-lane cards). Use when the user says \"plan the waves\", \"plan parallelization\", \"pwo plan\", \"PWO S1\", or after the harness probe, before the first create-story."
---

# pwo-plan-waves

Transform a BMad backlog (epics + architecture) into a **safe parallelization playbook** —
the project instance of the parallel-wave method, consumed downstream by `pwo-build-phase0`
(S2) and `pwo-run-wave` (S3). You run **once per project**, after the harness probe
(`pwo-probe-harness`, S0) and before the first create-story (see *Re-planning mid-build* for the
one scoped exception). You are an **orchestrator**:
you delegate the bulky dependency analysis to a fan-out of subsystem agents and keep only the
load-bearing judgement (the DAG, the wave cut, the gate), and you bring the two load-bearing
decisions — *should we even parallelize* and *is the wave plan sound* — to the human.

**The two non-negotiables:**

1. **The applicability gate (P1/P2 → GO/NO-GO).** Parallelize **only if** P1 (a disciplined
   architecture) **and** P2 (whole-app mockups *and* a verified smoke harness) **and** a real
   DAG width ≥ ~4–5 **and** a large backlog. A NO-GO is **a useful decision in itself** — it
   says *build sequentially*, and you stop there. The gate is the human's call.
2. **The adversarially-verified DAG.** A wrong DAG is a **silent corruption risk**, not a
   cosmetic one — a false edge wastes width, a hidden edge silently lengthens the critical
   path or reddens a post-merge gate. So the DAG is hunted adversarially (refute every edge;
   hunt every hidden edge from real consumed symbols), never asserted from prose.

And the load-bearing precondition: you **refuse to freeze the mechanism** unless S0's
`harness-profile` exists and every applicable Fact is `clean` (or its escalation is resolved) —
the mechanism you plan (emulate-vs-invoke, union policy, concurrent worktrees) is only as sound
as that empirical profile.

## Resolution rules

- Bare paths and `{skill-root}` (e.g. `references/playbook-assembly.md`) resolve from this skill's installed directory.
- `{project-root}` → the project working directory.
- `pwo-plan-waves` → the skill directory's basename.

## On Activation

Human-triggered, single-pass: no memlog, no resume, no customization. The playbook is a
regenerable, hand-editable artifact — re-run to regenerate it; it is the single source S2/S3
consume. Talk to the human in `{communication_language}`.

1. **Config.** Read `{project-root}/_bmad/config.yaml` (and `.user.yaml` if present), defaults
   for anything missing. Resolve `{project-name}` (else the repo folder name), `{output_folder}`
   (else `{project-root}/_bmad-output`), `{communication_language}`, `main_branch`
   (else `git rev-parse --abbrev-ref HEAD`), `smoke_harness`, and `worktree_workspace`.
2. **Inputs** (ask the human for any whose location you cannot resolve — do not invent):
   - **harness-profile** — `{output_folder}/pwo/harness-profile.md` (S0's output).
   - **epics** — the backlog of stories.
   - **architecture / spine** — the layers/ports/invariants doc (the P1 evidence).
   - **project-context** — conventions, registries, the canonical types.
   - **mockups (P2)** — the whole-app reference screens (the P2 evidence + the smoke source-of-truth).
   - **output path** — default `{output_folder}/pwo/playbook.md`.

## Refuse without a clean harness-profile

Read the harness-profile. If it is **absent**, or `overall ≠ clean` with **any applicable
Fact** still non-`clean` and its escalation block unresolved, **stop and escalate** (AskUserQuestion):
state which Fact blocks you and why S1 cannot freeze the mechanism on it, and offer
**re-run S0 · resolve the escalation · accept-with-documented-risk · abort**. Do not silently
proceed — the union policy, the emulate-vs-invoke default, and the concurrent-worktree cut all
key off this profile. Record the resolution in the playbook's provenance block.

## Run the applicability gate (P1/P2 → GO/NO-GO)

This is the first gate of planning and the human owns the verdict. Read
`references/playbook-assembly.md` § *Applicability gate* for the full criteria, then assess and
present, with **evidence, not assertion**:

- **P1 — disciplined architecture:** unidirectional layers, a pure domain, ports/interfaces,
  single-source derived values, centralized registries. Cite the architecture. (P1 is what makes
  lanes disjoint and hotspots rare; without it the serial integration explodes.)
- **P2 — mockups + a verified smoke harness:** whole-app mockups exist *and* the `smoke_harness`
  capability is **verified as a probe** (cross-check the harness-profile / S5's target). A green
  CI renders no pixels — P2 is the only counter-power to the UI blind spot.
- **Width & size:** a *preliminary* width from a quick read of the epics' independent roots
  (≥ ~4–5) and a backlog large enough that wall-clock matters.

Present the GO/NO-GO via **AskUserQuestion**. On **NO-GO**, write the verdict + its reasons to
the playbook artifact (the decision *is* the deliverable) and **stop — build sequentially**. On
**GO**, proceed; note that the width here is preliminary and the next step confirms it (a
collapse below ~4–5 is a late NO-GO signal to bring back to the human).

## Analyze dependencies adversarially (the fan-out)

Delegate the analysis; keep the DAG judgement. Read `references/dependency-analysis.md` — it
holds the subsystem-partition guidance, the prompt construction, both StructuredOutput schemas,
the false-edge/hidden-edge rubric, and the hotspot taxonomy. Run it as a **Workflow** (your
invocation of this skill is the opt-in), mirroring S0's discipline: every subagent is forced to
the schema and to **ground truth — cite the real producing symbol, never guess; an edge with no
producing-symbol is a false-edge candidate, a consumed symbol with no on-DAG producer is a hidden
edge**.

The shape (author the SUBSYSTEMS + prompts from the reference; a barrier sits between discovery
and verification because hidden-edge hunting needs the global symbol map):

```js
export const meta = { name:'pwo-dep-analysis', description:'Adversarial blocked-by DAG + hotspots', phases:[{title:'Discover'},{title:'Verify'}] }
const SUBSYSTEMS = [ /* one per epic/layer/folder owner, authored from the reference */ ]
// Stage 1 (parallel): each subsystem proposes blocked-by edges (with producing-symbol EVIDENCE) + footprint + hotspots.
const found = (await parallel(SUBSYSTEMS.map(s => () =>
  agent(discoverPrompt(s), { label:`discover:${s.id}`, phase:'Discover', schema:EDGE_SCHEMA, effort:'high' })))).filter(Boolean)
const graph = assembleGraph(found)   // BARRIER, plain code: global symbol map + dedup + CYCLE DETECTION (hidden-edge hunting needs the whole map)
// Stage 2 (parallel): refute each edge (false? dissolvable by a hoisted stub?) + one COVERAGE verifier
// per story (mandatory for zero-edge stories) hunting the edges nobody proposed, against the global map.
const verified = (await parallel([
  ...graph.edges.map(e => () =>
    agent(refutePrompt(e, graph), { label:`refute:${e.child}<-${e.parent}`, phase:'Verify', schema:VERDICT_SCHEMA, effort:'high' })),
  ...graph.stories.map(s => () =>
    agent(coveragePrompt(s, graph), { label:`cover:${s.key}`, phase:'Verify', schema:COVERAGE_SCHEMA, effort:'high' })),
])).filter(Boolean)
return { graph, verified }
```

Fold the verdicts into the final **blocked-by DAG** + the **hotspot list** (registries,
migrations, glyph/icon maps, co-composed screens, schema/serialization points) — with the
reference's **completeness assertion** (every edge and every story carries exactly one verdict; a
dead agent is a re-run, not a gap), the **uncertain-keeps-the-edge** rule, and a final
**acyclicity check** on the folded graph. Record each **refuted false edge** (recovered width),
each **found missed/hidden edge** + how you neutralize it (hoist a contract in Phase 0), and each
**duplication cluster** (≥2 lanes needing one unowned computation → designate one producer or a
Phase-0 shared helper). Re-confirm the real width ≥ ~4–5; if it collapsed, return to the human
(late NO-GO).

## Cut the waves (the 3 forces)

Group the DAG into waves under the three forces, in order (see `references/playbook-assembly.md`
§ *Wave cut*): **(1) the DAG** — the hard bound; a child never precedes its parent, and the
critical-path depth is your wave-count floor and makespan floor. **(2) coherence-to-smoke** —
what *groups* lanes is what forms a demonstrable increment together; each wave closes on
something you can validate. **(3) the ~10-lane cap** — what *cuts* a coherent increment that
grew too wide; split it, each half keeping its own coherence point.

Tag each wave's type and its **end-of-wave gate**:

- **Foundation-wave** (pure engines, nothing on screen) → gate = **jest + the empirical sweep**
  (name the range **and the runner** in the playbook).
- **Screen-wave** (composition) → gate = **smoke-vs-mockups, delegated to S5** (`pwo-ui-smoke`).
  Give the wave a **smoke criterion** carrying exactly S5's inputs: the **journey** to play, the
  **reference mockup(s)**, and the **expected live figures**. (Note each UI lane's nav entry
  point — a lane whose entry lands in a later wave is render-deferred; let coherence-to-smoke
  pull "what makes a reachable screen" into one wave.)
- **Mixed wave** (force 2 pulled engines into a screen increment — the common case on a
  mockup-driven app) → **both** gates: the sweep over the merged result *and* the smoke criterion.
  The gate follows the **content, not the tag**.

Then run the two deterministic cut checks from `references/playbook-assembly.md` § *Wave cut*:
the **pairwise disjointness check** (path-prefix footprint intersections per wave — every
non-empty intersection is a declared hotspot with a protocol, else re-scope/re-cut) and the
**hotspot editor cap** (~2 concurrent editors per hotspot per wave unless flat AND guarded).

## Spec Phase 0

Produce the Phase 0 spec S2 will implement and prove (full checklist in
`references/playbook-assembly.md` § *Phase 0 spec*): the **migration-set guard** (S2 proves it
RED by fault injection) with the **pre-allocated migration-version table**; the **seams to
hoist** — dissolved real edges AND neutralized hidden edges — each with the consumer's
**position/ordering constraint** (esp. FK ordering — a positionally-wrong seam actively
misleads, FN-7/A1), its **semantic contract**, and a **contract test committed skipped**; the
**shared helpers to single-source** (and optionally the shared test fixtures/builders) from the
duplication clusters; the **`.gitattributes`** policy (`merge=union` only on flat registries,
never the state file or a structured/test file — S2 proves it **binds**); the **native deps to
pre-provision** (peer/transitive + export surface verified first); and the **test-ids split per
screen** (`test-ids/<screen>.ts` + barrel — lever 1, the one-time fix that kills the whole
union-corruption class).

## Write the per-lane cards

One card per story (schema + tier rules in `references/playbook-assembly.md` § *Per-lane cards*):
**footprint** (the files/folders it may touch — keep lanes disjoint), the **hotspots + protocol**,
the **⚙ anti-collision constraint *verbatim*** (this is what prevents collisions — it must land
verbatim in the story spec; anchor it to the canonical in-repo type, "follow `<type>`; the model
wins where this prose differs"), and a **tier** — `trivial` (pure engine) · `standard` (compose) ·
`critical` (money/barème | FK/cascade | migration | new native dep | fills a hoisted seam). The
tier is *pre-placed*, not a fine effort-table: it arms the gate's anti-mis-tiering guard and bumps
`critical` lanes to `dev=max`.

For every lane whose hotspots include the **migration-set**: **allocate the concrete version
number now**, in DAG-then-wave order, and bake it into the ⚙ constraint ("your migration is
version N — exactly N, do not compute it"). Concurrent lanes each computing `DB_VERSION+1` is the
deterministic collision the guard then catches *late*, at integration, forcing a hand-renumber
under pressure — pre-allocation kills it at the source. Record the allocation table (with its
baseline `DB_VERSION` at plan freeze, so a Phase-0 or gate-amendment migration shifts the whole
table mechanically) in the playbook's Phase 0 spec — S2's guard and S3's §7 verify against it.

## Assemble the playbook and validate it

Write the artifact from `assets/playbook-template.md` to the resolved output path: provenance
(inputs + the harness-profile dependency check), the applicability-gate verdict, the DAG (+ the
refuted false edges / found hidden edges), the hotspots + protocols, the wave plan (each wave's
type + end-of-wave gate/smoke criterion), the Phase 0 spec, and the per-lane cards.

Before bringing it to the human, run the closing assertions: the **per-wave disjointness check**
passes (or every intersection is a protocol-carrying hotspot); **the set of edges removed as
dissolvable matches the Phase 0 seam list one-to-one** (a dissolved edge whose seam never reaches
S2 resurfaces as an in-lane improvised stub); the folded DAG is **acyclic**; and every
**duplication cluster** has a designated producer or a Phase-0 shared-helper entry.

Then **bring the wave plan to the human for validation before S2/S3** (AskUserQuestion): the
wave cut, the critical path, and the Phase 0 spec. **Cross-check every load-bearing value**
(seeds, barèmes, fiscal constants the cards reference) against the source of truth yourself — do
not let a card carry an unverified number into a `critical` lane. Record the human's sign-off in
the playbook. The validated playbook is the hand-off to `pwo-build-phase0` (S2).

## Re-planning mid-build (scoped re-entry)

The playbook is regenerable, and a build can prove it wrong — a missed edge surfacing at wave k, a
legitimate refactor that moved a card's anchor type. When S3 escalates a plan defect, re-run this
skill in **scoped re-entry**: pin every integrated wave as fact (from the run-records + real git),
re-run the adversarial analysis **only over the unmerged stories against the CURRENT base** (where
on-base evidence now genuinely exists), preserve the recorded gate amendments, and re-cut only the
remaining waves. Record the re-plan in the playbook's provenance block. Never regenerate from
scratch mid-build — that discards the amendments and the merged reality.

## Hand off to the next step

PWO is a **guided pipeline**: each step ends by handing the user a paste-ready prompt for the next one,
run in a **fresh session**. **Branch on the gate verdict:**

- **NO-GO** (or width collapsed below ~4–5) → emit **no** handoff. The decision *is* the deliverable:
  the project builds **sequentially**, so tell the user plainly that PWO stops here and to proceed with
  the normal `create-story` / `dev-story` flow.
- **GO and the wave plan is signed off** → emit — in **English** — a single fenced code block whose
  **first token is the next command** followed by a self-contained prompt, then tell the user: **"Copy
  this into a NEW Claude Code session (fresh context) to run it."** Resolve every `{token}` to its real
  value (the playbook path, `main_branch`, the project name). Skip in headless mode (the JSON tail
  carries `recommendation` + `playbook`).

```
/pwo-build-phase0 The wave plan is validated for {project-name} (applicability gate: GO) — playbook at
{output_folder}/pwo/playbook.md. Build AND prove the Phase 0 guard-rails on {main_branch} from the
playbook's "Phase 0 spec (for S2)" section: the migration-set guard (prove it RED by fault injection),
the hoisted cross-wave seams (FK/ordering constraints + semantic contracts baked in, contract tests
committed skipped), the shared helpers to single-source (real impls), the .gitattributes merge policy
(prove it BINDS: check-attr + one live merge trial), the pre-provisioned native deps, the per-screen
test-ids split, and the worktree workspace + state-file ownership policy. Write the receipt to
{output_folder}/pwo/phase-0-receipt.md. Fresh session.
```

## Headless output

When invoked headless, skip the interactive gates: still assemble the playbook, mark the
applicability GO/NO-GO and the wave-plan sign-off `pending-human` (with your *recommended*
verdict + evidence), and return JSON only —
`{ "status": "complete" | "blocked", "recommendation": "GO" | "NO-GO", "playbook": "<path>", "width": <n>, "waves": <n>, "pendingHuman": ["applicability-gate", "wave-plan", …] }`
(`"blocked"` with a one-line `reason` if the harness-profile is absent/non-clean — that
precondition is never waived headlessly).
