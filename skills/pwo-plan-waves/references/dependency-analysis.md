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
  hunting needs the *whole* map.
- **Dedup the edge set**; drop self-edges; collapse duplicate `child←parent` pairs.
- **Resolve `consumesUnresolved`**: for each, look it up in the global map. If a producer exists
  → it was a **missed edge**, add it. If none exists → it's a **hidden-edge candidate** (the
  symbol is assumed but nobody on the DAG makes it) → flag for the verification stage and for
  Phase 0 (a seam to hoist, or a real gap to escalate).

## Stage 2 — adversarial verification (parallel, one agent per edge)

This is the non-negotiable. Each agent is handed one edge (and the graph context) and is
prompted to **try to refute it** — default-to-skeptical, because a false edge silently wastes
width. Separately it hunts hidden edges around the same story pair. The schema:

```js
const VERDICT_SCHEMA = { type:'object', additionalProperties:false, properties:{
  child:{type:'string'}, parent:{type:'string'},
  edgeReal:{type:'boolean'},                                   // false => false edge, REMOVE it (recovers width)
  refutationAttempt:{type:'string'},                           // how you tried to break it; what you found in the real code
  dissolvableByStub:{type:'boolean'},                          // true => the edge can be cut by hoisting a typed no-op stub in Phase 0
  stubProposal:{type:'string'},                                // the seam to hoist, if dissolvable (port + the consumer's position/ordering constraint)
  hiddenEdges:{type:'array', items:{type:'object', additionalProperties:false, properties:{
    child:{type:'string'}, neededSymbol:{type:'string'}, producerKey:{type:'string'}, evidence:{type:'string'} },
    required:['child','neededSymbol','producerKey','evidence'] }},
  notes:{type:'string'} },
  required:['child','parent','edgeReal','refutationAttempt','dissolvableByStub','stubProposal','hiddenEdges','notes'] }
```

`refutePrompt(e, graph)` instructs the agent to: open the real code at the consumed symbol; try
to prove the child does **not** actually need the parent merged first (the symbol already exists
on the base, or the child only needs an interface that can be hoisted); decide `edgeReal`; if
real but `dissolvableByStub`, propose the seam to hoist **with the consumer's position/ordering
constraint** (esp. FK ordering — FN-7/A1: a positionally-wrong seam actively misleads); and
hunt `hiddenEdges` (does the child consume anything whose only producer is in a *later* wave?).

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

Fold the verdicts: remove false edges, dissolve stub-able ones (record the seam for Phase 0),
add confirmed hidden edges (with their neutralization). The result is the final blocked-by DAG.

## Hotspot taxonomy (every multi-lane file, tagged + given a protocol)

A hotspot is any file more than one lane edits. Tag each `kind` and assign the `protocol` the
per-lane cards will carry:

| kind | examples | protocol |
| ---- | -------- | -------- |
| `flat-registry` | append-only barrels, a flat `export const` list | `merge=union` safe *only if* truly flat; else split |
| `structured-registry` | `test-ids.ts` (trailing `type`/aggregate), multi-line `as const` blocks | **split per screen** (lever 1); never union; if unavoidable, hand-resolve + `tsc` backstop |
| `contract-test` | the registry's `.test.ts` (multi-line `it()` blocks) | **never union**; hand-resolve (partial-block close, then strip markers) |
| `migration-set` | the migrations file / `DB_VERSION` | one new object at the **pre-allocated** version, ascending; the guard catches dups |
| `glyph-map` | a shared `Icon.tsx` / Lucide glyph object | keep-both on the import list + the glyph object; manual merge after the first lane |
| `co-composed-screen` | a screen N lanes add children to | **single-owner shell** + each non-owner adds only its own child slot, preserves the root testID |
| `shared-store` | a store N lanes extend | classify each hunk (keep-both interface/impl vs keep-HEAD singleton); never blanket-strip |
| `schema/serialization` | round-trip mappers, `assembleInput` helpers | single source; flag a consolidation story if N lanes re-implement it |

Always include the **test-id/registry file AND its contract test AND any shared glyph map** in
the hotspot list — these conflict on essentially every multi-lane integration (method FN-2/4/5/7).
The structured-registry and contract-test pains are exactly what Phase 0's **test-ids split**
(lever 1) is designed to eliminate at the source.

## Extending the analysis (new hotspot kinds, deeper verification)

To add a hotspot kind or a sharper refutation lens, copy the shapes above: a `kind` + a
deterministic `protocol` for the taxonomy, or another skeptical question in `refutePrompt`. Keep
the ground-truth rule and the schemas stable — the orchestrator folds whatever this file
declares, the same way `pwo-probe-harness` folds whatever its catalog declares.
