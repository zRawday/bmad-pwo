# Dependency Analysis — the adversarial fan-out

This file is the delegated half of `pwo-plan-waves`: how to derive the **blocked-by DAG** and
the **hotspots** by fanning out subsystem agents and then **adversarially verifying** the
result. SKILL.md reads this to author the discovery prompts, the refutation prompts, and the two
StructuredOutput schemas. The discipline mirrors `pwo-probe-harness`: every agent reports
**ground truth from the real code**, never a plausible guess — because a wrong DAG is a silent
corruption risk, not a cosmetic one.

## The non-negotiable every analysis agent carries

Bake these into every discovery and refutation prompt, verbatim in spirit:

- **Cite the real producing symbol, never guess.** An edge `child blocked-by parent` exists
  only if `child` consumes a concrete symbol/file/migration/seam that `parent` produces — and
  you must name it (the export, the table, the testID file, the migration version) with its
  path. **An edge with no captured producing-symbol is a false-edge candidate**, not an edge.
- **A consumed symbol with no on-DAG producer is a hidden edge.** If a story imports/calls
  something that no listed story produces (it's assumed to "already exist" but doesn't on this
  story's base), that is a hidden dependency — the most dangerous kind, because it reddens a
  post-merge gate silently.
- **Read the code, not the prose.** When a story's spec prose and the canonical in-repo type
  disagree, the **shipped type wins**; report the mismatch. (Method FN-4: a dev follows shipped
  code over loose prose — so the DAG must too.)
- **Ground-truth or nothing.** `evidence` holds the literal symbol + path you found; the
  determination derives only from it. No evidence ⇒ say so; do not infer an edge from topic
  similarity ("both touch fiscal" is not a dependency).

## Partition into subsystems

Author one discovery agent per **subsystem** — the natural ownership unit, in this preference
order: the epic, then the architectural layer (domain / data / store / ui), then the folder
owner. Aim for subsystems that are mostly disjoint so each agent reasons locally; cross-subsystem
edges are exactly what the verification stage hunts. Give every agent the **full story list**
(so it can name a producer in another subsystem) but ask it to propose edges only for the
stories it owns. Keep subsystems ≤ ~8 agents; a very large backlog can run discovery in two
rounds.

## Stage 1 — discovery (parallel, one agent per subsystem)

Each agent reads the epics + architecture + the real code for its subsystem and returns, per
owned story: its proposed `blocked-by` edges (each with producing-symbol evidence), its
**footprint** (files/folders it will touch), and the **hotspots** it touches (from the taxonomy
below). The shared schema:

```js
const EDGE_SCHEMA = { type:'object', additionalProperties:false, properties:{
  subsystem:{type:'string'},
  stories:{type:'array', items:{type:'object', additionalProperties:false, properties:{
    key:{type:'string'}, title:{type:'string'},
    footprint:{type:'array', items:{type:'string'}},
    producesSymbols:{type:'array', items:{type:'string'}},     // exports/tables/migrations/seams THIS story creates (path-qualified)
    blockedBy:{type:'array', items:{type:'object', additionalProperties:false, properties:{
      parentKey:{type:'string'}, consumedSymbol:{type:'string'}, evidence:{type:'string'} },  // the literal symbol + where it is consumed
      required:['parentKey','consumedSymbol','evidence'] }},
    consumesUnresolved:{type:'array', items:{type:'string'}},  // symbols it consumes but cannot attribute to ANY listed story (hidden-edge candidates)
    hotspots:{type:'array', items:{type:'object', additionalProperties:false, properties:{
      file:{type:'string'}, kind:{type:'string'}, protocol:{type:'string'} },
      required:['file','kind','protocol'] }} },
    required:['key','title','footprint','producesSymbols','blockedBy','consumesUnresolved','hotspots'] }} },
  required:['subsystem','stories'] }
```

`discoverPrompt(s)` instructs the agent to: read its subsystem's stories + the architecture +
the real code; for each owned story enumerate `producesSymbols` (path-qualified) and `blockedBy`
(each with the consumed symbol + evidence); list `consumesUnresolved` (anything it needs that no
listed story produces); and tag every multi-lane file as a hotspot with its protocol. Carry the
non-negotiable verbatim.

## Barrier — assemble the graph (plain code, not an agent)

Fold the discovery returns deterministically:

- **Build the global symbol map** `produces: symbol → producing story` from every story's
  `producesSymbols`. This is why discovery is a barrier before verification — hidden-edge
  hunting needs the *whole* map. **A symbol with ≥2 producing stories is a plan-time duplication
  signal**, never a map entry to last-write-win: designate ONE producer (edge or wave-order the
  others onto it) or hoist the helper in Phase 0, and record the designation on every consumer's
  card.
- **Dedup the edge set**; drop self-edges; collapse duplicate `child←parent` pairs.
- **Detect cycles** (strongly-connected components) after dedup — a cycle makes the wave cut's
  hard bound unsatisfiable, and it is **never resolved silently**. For each: prefer dissolving one
  direction via a hoisted Phase-0 stub (record the seam); else merge the stories into one lane
  (re-card it — combined footprint, the higher tier; reserve this for hard-hard cycles); else
  escalate to the human. Record every cycle + its resolution in the playbook's DAG section, and
  **re-check acyclicity after the verdict fold** (a confirmed missed edge can close a new cycle).
- **Fold the hotspot lists** — the same file reported by N agents is ONE hotspot; its `protocol`
  comes from the taxonomy by `kind`; two agents disagreeing on the kind is resolved by inspecting
  the real file, never by keeping both strings.
- **Resolve `consumesUnresolved`**: for each, look it up in the global map. If a producer exists
  → it was a **missed edge**, add it. If none exists → it's a **hidden-edge candidate** (the
  symbol is assumed but nobody on the DAG makes it) → flag it for the per-story coverage pass and
  for Phase 0 (a seam to hoist, or a real gap to escalate).

One fold is judgement, not plain code, so it is yours (the orchestrator's), right after the
barrier: **cluster what the stories *compute***, not just what they name. Read every story's
`producesSymbols` + AC verbs and group the computations ≥2 lanes will need even when the names
differ ("assemble the frais-réels input", written three ways). Every cluster with no single owner
is a plan-time duplication signal: designate one producing story (and edge/wave-order the
consumers onto it), or list the small pure helper in the Phase 0 spec's *Shared helpers* item;
either way every consumer card carries the single-source constraint ("import it, never
re-implement"). Deferring the cluster to the end-of-wave critic means N green copies have already
shipped.

## Stage 2 — adversarial verification (parallel: one agent per edge, one per story)

This is the non-negotiable, and it runs **two passes together**: a **per-edge refutation** and a
**per-story coverage** pass. The refuters are skeptical of *edges*, never of *safety* — the error
costs are asymmetric. A wrongly-kept false edge costs width (recoverable). A wrongly-removed real
edge schedules the child beside its parent; the child's dev, unable to find the missing symbol,
re-implements it under footprint — green, silent, merged duplication. So **removal requires
positive, cited evidence** (the consumed symbol already on the child's base at a named path, or a
cited spec/AC line proving the child never reads the parent's output); *failure to confirm is not
refutation*; **when uncertain, KEEP the edge**. Each edge agent is handed one edge (and the graph
context) and prompted to **try to refute it**. Separately it hunts hidden edges around the same
story pair. The schema:

```js
const VERDICT_SCHEMA = { type:'object', additionalProperties:false, properties:{
  child:{type:'string'}, parent:{type:'string'},
  edgeReal:{type:'boolean'},                                   // false => false edge, REMOVE it — ONLY with cited on-base evidence
  confidence:{type:'string', enum:['certain','probable','uncertain']},  // 'uncertain' => the edge STAYS, whatever edgeReal says
  refutationAttempt:{type:'string'},                           // how you tried to break it; what you found in the real code
  dissolvableByStub:{type:'boolean'},                          // true => the edge can be cut by hoisting a typed no-op stub in Phase 0
  stubProposal:{type:'string'},                                // the seam to hoist, if dissolvable: port + the consumer's position/ordering constraint + the SEMANTIC contract (units, rounding/precision, sign, empty/error behavior — derived from the CONSUMER's spec/ACs)
  hiddenEdges:{type:'array', items:{type:'object', additionalProperties:false, properties:{
    child:{type:'string'}, neededSymbol:{type:'string'}, producerKey:{type:'string'}, evidence:{type:'string'} },
    required:['child','neededSymbol','producerKey','evidence'] }},
  notes:{type:'string'} },
  required:['child','parent','edgeReal','confidence','refutationAttempt','dissolvableByStub','stubProposal','hiddenEdges','notes'] }
```

`refutePrompt(e, graph)` instructs the agent to: open the real code at the consumed symbol; try
to prove the child does **not** actually need the parent merged first (the symbol already exists
on the base, or the child only needs an interface that can be hoisted); decide `edgeReal` **with
`confidence`** — an `edgeReal:false` must cite the on-base symbol (path) or the spec/AC line that
proves the child never reads the parent's output, and carry the tie-break **verbatim**: *"when
uncertain, KEEP the edge — a kept false edge costs width; a removed real edge is a silent
duplication seed"* (note the parent's output usually does **not** exist in the repo yet — absence
of code is not evidence of a false edge; the map of planned `producesSymbols` is); if real but
`dissolvableByStub`, propose the seam to hoist **with the consumer's position/ordering
constraint** (esp. FK ordering — FN-7/A1: a positionally-wrong seam actively misleads) **and its
semantic contract**; and hunt `hiddenEdges` (does the child consume anything whose only producer
is another listed story that is not already an ancestor of this child? — waves do not exist yet
at this stage).

### The per-story coverage pass (the edges nobody proposed)

Refuting only *proposed* edges leaves the worst class unexamined: the edge **nobody proposed**. A
story that came back with `blockedBy:[]` and an empty `consumesUnresolved` has never been
challenged — its discovery agent may simply not have *noticed* a consumption, and an unnoticed
consumption never reaches the barrier's map. So stage 2 also runs **one coverage verifier per
story** (mandatory at minimum for every zero-edge story and every story the barrier flagged),
handed the story's spec/ACs **plus the full global `producesSymbols` map**: enumerate everything
this story will consume (symbols, tables, seams — re-derived from the ACs, not copied from the
card), check each against the map, and return missed edges keyed by *story coverage*, not by
proposed-edge adjacency. This pass is the designated consumer of the barrier's
hidden-edge-candidate flags. The schema:

```js
const COVERAGE_SCHEMA = { type:'object', additionalProperties:false, properties:{
  storyKey:{type:'string'},
  consumesChecked:{type:'array', items:{type:'string'}},       // everything the story will consume, enumerated from the ACs
  missedEdges:{type:'array', items:{type:'object', additionalProperties:false, properties:{
    child:{type:'string'}, neededSymbol:{type:'string'}, producerKey:{type:'string'}, evidence:{type:'string'} },
    required:['child','neededSymbol','producerKey','evidence'] }},
  notes:{type:'string'} },
  required:['storyKey','consumesChecked','missedEdges','notes'] }
```

`coveragePrompt(s, graph)` instructs the agent to: read the story's spec/ACs + the architecture;
enumerate `consumesChecked` from the ACs (not from the card); look each item up in the global
`producesSymbols` map; report every consumption whose producer is another listed story that is not
already this story's ancestor as a `missedEdges` entry (with evidence); and carry the ground-truth
non-negotiable verbatim.

## The false-edge / hidden-edge rubric

- **False edge** (`edgeReal:false`) — the consumed symbol already exists on the child's base, or
  the "dependency" is topical not structural (both touch the same domain but share no symbol), or
  the parent's output is not actually read by the child. **Remove it — it was wasting width.**
- **Real but dissolvable** (`edgeReal:true, dissolvableByStub:true`) — the child needs only an
  *interface*, not the parent's implementation. **Hoist a typed port + no-op stub in Phase 0** so
  the child binds early; the edge leaves the DAG (the producer fills the impl later). This is how
  you keep the critical path short.
- **Real and hard** (`edgeReal:true, dissolvableByStub:false`) — a genuine
  produce-then-consume (a migration the child's schema needs, a table the child writes). **Keep
  it**; it constrains the wave order.
- **Hidden edge** — a consumed symbol whose only producer is downstream. **The most dangerous:**
  it doesn't show until a post-merge gate goes red. Neutralize by **hoisting the contract in
  Phase 0** (preferred) or by **reordering** so the producer precedes the consumer; if neither is
  possible it's a real backlog gap → escalate to the human.

Fold the verdicts with a **completeness assertion**: every proposed edge and every story carries
exactly one verdict — a dead agent's silently-dropped return (`.filter(Boolean)`) is a re-run, not
a gap. Remove false edges **only where the removal cites on-base evidence and `confidence ≠
uncertain`**; dissolve stub-able ones (record the seam for Phase 0); add confirmed missed/hidden
edges (with their neutralization); re-verify the folded graph is **acyclic**. The result is the
final blocked-by DAG.

## Hotspot taxonomy (every multi-lane file, tagged + given a protocol)

A hotspot is any file more than one lane edits. Tag each `kind` and assign the `protocol` the
per-lane cards will carry:

| kind | examples | protocol |
| ---- | -------- | -------- |
| `flat-registry` | append-only barrels, a flat `export const` list | `merge=union` safe *only if* truly flat; else split |
| `structured-registry` | `test-ids.ts` (trailing `type`/aggregate), multi-line `as const` blocks | **split per screen** (lever 1); never union; if unavoidable, hand-resolve + `tsc` backstop |
| `contract-test` | the registry's `.test.ts` (multi-line `it()` blocks) | **never union**; hand-resolve (partial-block close, then strip markers) |
| `migration-set` | the migrations file / `DB_VERSION` | one new object at the **pre-allocated** version, ascending; the guard catches dups (S1 allocates the concrete numbers on the cards — see SKILL.md § per-lane cards) |
| `glyph-map` | a shared `Icon.tsx` / Lucide glyph object | keep-both on the import list + the glyph object; manual merge after the first lane |
| `co-composed-screen` | a screen N lanes add children to | **single-owner shell** + each non-owner adds only its own child slot, preserves the root testID |
| `shared-store` | a store N lanes extend | classify each hunk (keep-both interface/impl vs keep-HEAD singleton); never blanket-strip |
| `schema/serialization` | round-trip mappers, `assembleInput` helpers | single source; record it in the playbook's **Planned consolidations (for S4)** section if N lanes must re-implement it until all consumers exist |

Always include the **test-id/registry file AND its contract test AND any shared glyph map** in
the hotspot list — these conflict on essentially every multi-lane integration (method FN-2/4/5/7).
The structured-registry and contract-test pains are exactly what Phase 0's **test-ids split**
(lever 1) is designed to eliminate at the source.

## Extending the analysis (new hotspot kinds, deeper verification)

To add a hotspot kind or a sharper refutation lens, copy the shapes above: a `kind` + a
deterministic `protocol` for the taxonomy, or another skeptical question in `refutePrompt`. Keep
the ground-truth rule and the schemas stable — the orchestrator folds whatever this file
declares, the same way `pwo-probe-harness` folds whatever its catalog declares.
