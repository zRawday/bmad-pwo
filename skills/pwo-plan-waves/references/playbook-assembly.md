# Playbook Assembly — the orchestrator's synthesis

This file is the orchestrator's half of `pwo-plan-waves`: the judgement SKILL.md keeps in its own
context (the gate criteria, the wave cut, the Phase 0 checklist, the card schema) after delegating
the bulky dependency analysis. It does **not** restate the fan-out — that lives in
`dependency-analysis.md`. Read the section you need when SKILL.md points here.

---

## Applicability gate

Parallelization only pays back when the architecture keeps lanes disjoint **and** the UI can be
validated as you go. Assess all four, present **evidence not assertion**, and let the human make
the GO/NO-GO call. A NO-GO is a real, useful output: *build sequentially* — write it to the
playbook and stop.

### P1 — a disciplined architecture (the lane-disjointness precondition)

Check the architecture/spine for, and cite, each signal:

- **Unidirectional layers** (domain ← data ← store ← ui; no upward imports).
- **A pure domain** — derived values are computed, never persisted (so two lanes can't disagree
  about a stored derivation).
- **Ports / interfaces** at the seams (so a consumer binds to a contract, not an impl — this is
  what makes hidden edges *dissolvable* by a hoisted stub).
- **Single-source derived values** (one home per predicate/formula; no silent re-implementation).
- **Centralized registries** (testIDs, glyphs) — rare, mechanical hotspots rather than ad-hoc.

The cleaner P1, the **rarer and more mechanical** the hotspots, and the cheaper the serial
integration. On a weak P1 the hotspots are everywhere and the serial resolution explodes — that
alone can flip the ROI to NO-GO.

### P2 — whole-app mockups + a verified smoke harness (the UI-blind-spot counter-power)

A green CI renders no pixels; jest with fakes proves logic, never the screen. P2 is the only
counter-power. It requires four things:

- **(a)** mockups/screens for the **whole** app — the visual source of truth to gate against;
- **(b)** a **smoke harness** fit for this project's target (Expo MCP / Chrome DevTools / adb…),
  its capability **verified as a probe** before betting on it (cross-check the harness-profile and
  `smoke_harness` / S5's target);
- **(c)** the smoke validates **coherence with the mockup**, not just absence of a crash;
- **(d)** each wave **closes on a smokable increment** (see the wave cut) — so validation happens
  continuously instead of accumulating render-deferred debt.

If `smoke_harness = none` or no mockups exist, P2 fails — screen-wave validation degrades to
jest-only and the UI blind spot stays uncovered. Surface that to the human as a NO-GO (or a
documented, accepted risk).

### Width & size

- **Width** — a *preliminary* count of independent roots from a quick epic read (≥ ~4–5). It is
  preliminary because real width is only known after the adversarial DAG; SKILL.md re-confirms it
  there and brings a collapse back as a late NO-GO.
- **Size** — a backlog large enough that wall-clock matters. A short, linear backlog never repays
  the orchestration overhead → NO-GO regardless of P1/P2.

GO requires **P1 ∧ P2 ∧ width ≥ ~4–5 ∧ large backlog**. Anything missing ⇒ NO-GO, build
sequentially. (Run standalone, this gate is also a useful decision tool on a candidate backlog.)

---

## Wave cut (the 3 forces)

A wave is **not** just "lanes whose DAG parents are merged" — that is necessary but insufficient.
Cut waves so each closes on a **coherent, smokable increment**. Three forces, in order:

1. **The DAG (hard bound).** A child never precedes its parent. The **critical-path depth is the
   wave-count floor and the makespan floor** — find the longest chain, protect it, never starve it
   to feed a shallow one. Keep it short with the hoisted contracts from Phase 0.
2. **Coherence-to-smoke (the grouping force).** What belongs together is what forms a demonstrable
   increment together — "what makes a reachable screen / a complete feature." This is what
   dissolves render-deferred debt: group "what makes an entry-reachable screen" into one wave so no
   UI lane hangs three waves unvalidated.
3. **The ~10-lane cap (the cutting force).** When a coherent increment grows past ~10 lanes, split
   it — each half keeping its own coherence point. Beyond ~10, three things derail together: the
   smoke becomes unmanageable, the serial integration (C1) explodes, the orchestrator context (the
   2nd budget) overflows. ~10 is where the method empirically peaked.

Do **not** cut to fill idle agents, to go fast, or to maximize reachability for its own sake — cut
so every boundary leaves the app in a coherent, validatable state.

And two checks every cut must pass before the playbook ships (both deterministic; run them, don't
eyeball them):

- **Pairwise disjointness.** For every wave, compute the pairwise footprint intersections of its
  lanes — with **path-prefix semantics** (`src/store/` ∩ `src/store/useX.ts` is non-empty).
  Every non-empty intersection must be a **declared hotspot carrying a protocol**; else re-scope
  the cards or re-cut the wave. Footprints are pre-spec *predictions* by different discovery
  agents: two lanes can innocently claim the same file with neither tagging it a hotspot.
- **Hotspot editor cap.** Cap the concurrent editors of any one hotspot within a wave at **~2** —
  unless the file is genuinely flat AND guarded. The module's own evidence says union corruption
  starts at the **3rd** concurrent appender (FN-2/5/7); spread hotspot-heavy lanes across waves.

### Wave types and their end-of-wave gate

- **Foundation-wave** — pure engines, nothing on screen. End-of-wave gate = **jest + the empirical
  sweep** (sweep the realistic input range; a green unit suite is not enough — method FN-4/5).
- **Screen-wave** — composition; something renders. End-of-wave gate = **smoke-vs-mockups,
  delegated to `pwo-ui-smoke` (S5)**. The wave carries a **smoke criterion** shaped as exactly
  S5's inputs, so the playbook hands S3 a ready call:
  - **journey** — the ordered steps to play (navigate here, enter X, expect screen Y);
  - **reference mockup(s)** — the P2 screen(s) the render must match;
  - **expected live figures** — the numbers to verify on a real frame (e.g. `indemnité km = 127,20 €`).

  Note each UI lane's **nav entry point**: if its entry lands in a later wave, its smoke is
  render-deferred to that host wave — let coherence-to-smoke pull the entry and the screen into the
  same wave wherever the DAG allows.

The tag names the wave's *dominant* content; **the gate follows the content, not the tag**. A wave
containing **any** engine/computation lane carries the **jest + empirical sweep over the merged
result** (name the swept range — and its runner — in the playbook) **in addition to** the S5 smoke
criterion when anything renders. Coherence-to-smoke deliberately builds mixed waves; a "screen" tag
must never silently drop the merged-composition sweep for the engines it pulled in — per-lane
review sweeps each engine in its own worktree, and only the post-merge sweep sees the composition.

---

## Phase 0 spec

The safety infra S2 (`pwo-build-phase0`) implements and **proves** on the main branch before Wave
1. Specify each item concretely (S2 builds exactly what this lists):

- **Migration-set guard** — assert versions are strictly increasing / unique / contiguous,
  `DB_VERSION == count == max`, and **no duplicate `CREATE TABLE` / `ADD COLUMN`** across the whole
  set. S2 must **prove it goes RED** by fault injection (blind spot A3: an untested guard does not
  count). Generalize it to any registry a bad merge can silently duplicate.
- **Seams to hoist** — for every edge the verification stage dissolved by a stub — **both** an
  `edgeReal:true / dissolvableByStub:true` edge **and** a neutralized hidden edge: a **typed port
  + a no-op stub** so the consumer binds early. **For each seam, record the consumer's
  position/ordering constraint** — especially **FK ordering** (under `foreign_keys=ON` with no
  cascade, a delete/purge must precede the referenced-row delete) — **and its semantic contract**
  (units, rounding/precision, sign, empty/error behavior, from the consumer's spec), plus an
  **executable contract test committed `skipped` in Phase 0** — un-skipped (skipped→green, i.e. it
  would have been RED against the no-op) when the producer fills the impl. A positionally-wrong
  seam is *more* dangerous than no seam: it actively misleads the dev, who follows shipped code
  over prose (FN-7/A1). S3's gate re-validates seam position against the actual consumer.
- **Shared helpers to single-source** — for each computation ≥2 lanes need and no story owns (the
  duplication clusters from the analysis): the default is to **designate ONE producing story** and
  edge/wave-order the consumers onto it, every consumer card carrying "import it, never
  re-implement"; where that ordering would starve the width, list the small **pure** helper here
  for S2 to implement once on main (a real impl with its unit test — a no-op stub of a *formula*
  forces every consumer to mock it, which defeats the point).
- **Shared test fixtures/builders** *(optional)* — when ≥2 lanes will seed the same entities,
  pre-provision one fixture-builder home on main and reference it from every consumer card — the
  test-ids move applied to the test layer: N hand-rolled seeders with subtly different assumptions
  (VAT-inclusive vs VAT-exclusive) each stay green in isolation and contradict each other at the
  first parity test.
- **`.gitattributes`** — `merge=union` **only** on genuinely flat registries; **never** on the
  coordination/state file a tool parses (union duplicates keys, breaks the parser), and **never**
  on a structured registry or a contract `.test.ts` (union splices silently — FN-2/5/7).
- **Native deps to pre-provision** — list every native dep the backlog needs (from the
  architecture/epics) so S2 installs them **once on main** before the waves, sparing the per-lane
  dep-shim dance (cost C3). **Verify the full peer/transitive + real export surface first** (FN-4:
  a v14 test lib needed a separate `test-renderer` peer; FN-1: an fs API had moved to `/legacy`).
- **Test-ids split per screen** — `test-ids/<screen>.ts` + a barrel, so each lane writes only its
  own file: **zero collision, zero union, zero manual resolution.** This is **lever 1**, the single
  serious technical fix — it closes the entire union-corruption class that recurred every wave
  (C1). List the per-screen split the test-id usage implies.
- **Worktree workspace + ownership policy** — the sibling `worktree_workspace` directory for the
  lanes, and the state-file ownership rule: **lanes never write the coordination file; the
  orchestrator owns it 100%.**

---

## Per-lane cards

One card per story — the unit S3's create step turns into a spec. Capture:

```
key            the story id
title          one line
footprint      the files/folders this lane may touch (keep lanes disjoint)
hotspots       the multi-lane files it touches + the protocol (from the taxonomy)
entryPoint     for a UI lane: its nav entry (and whether it's reachable this wave or deferred)
constraint     the ⚙ anti-collision constraint, VERBATIM (see below)
tier           trivial | standard | critical (derived — see below)
```

### The ⚙ anti-collision constraint (verbatim, load-bearing)

This is **what prevents collisions** — it must land **verbatim** in the story spec S3 authors.
Write it as the imperative the lane must obey: purity (`"this engine is pure — import nothing from
data/ui"`); the single-source claim (`"this is the single source of predicate Z — consumers import
it, never re-implement"`); the don't-touch (`"do NOT add a writer here; expose reusable action Y so
consumers bind to it"`); the hotspot protocol (`"append only your own block to <registry>; never
edit another lane's"`). **Point it at the canonical in-repo type** where prose might diverge:
`"follow <type>; the model wins where this prose differs"` (FN-4 — a dev follows the shipped type
over loose prose, and that is the safety net). A vague constraint loads the gate (a scarce budget);
a precise, code-anchored one is honored cleanly.

### Tier (pre-placed, not a fine effort-table)

Derive the tier from the lane's nature:

- **trivial** — a **pure engine** (pure domain, fully unit-checkable).
- **standard** — a **compose** lane (wires existing pieces into a screen/flow).
- **critical** — anything touching **money/barème**, an **FK/cascade**, a **migration**, a
  **new native dep**, or **filling a Phase-0 hoisted seam** (its consumers bound waves ago to a
  contract this lane must now honor exactly). These are where a silent defect is expensive (a
  wrong cent, an on-device FK throw, a corrupted schema, an API mismatch, a producer/consumer
  semantic drift).

The tier is **pre-placed**, not the seed of a `tier × task` effort-table — that finer table is
**deferred and data-driven** (measure `tier + effort + result` in the Field Notes; only build it
if a back-end-heavy project's data justifies it). Pre-placing the tier serves two concrete jobs
now: it arms the **anti-mis-tiering guard at the gate** (a `critical` lane spec that reads thin
gets caught), and it tells S3 to **bump `critical` lanes to `dev=max`** (tiering upward is the only
safe direction — main green > speed > cost; token economy is not a goal).
