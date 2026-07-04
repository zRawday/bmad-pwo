# Phase-0 Build — the per-item recipes

This file is the build half of `pwo-build-phase0`: the concrete how-to for each Phase-0 item the
SKILL.md spine points here for. Read the section you need when SKILL.md sends you here; do not read
the whole file to build one item. Each item is built from what the **playbook's `Phase 0 spec (for
S2)`** lists for *this* project — these recipes are the method, the spec is the instance. The
discipline shared across all of them: **the orchestrator stays thin** (delegate the bulky coding,
keep the load-bearing verification — #2), **main never goes red on the way in** (gate after every
commit — #6), and **every claim is checked against real git / real output**, never a self-report (#1).

---

## Delegating an item (the shared shape)

Phase 0 is code on the main branch, not lanes in worktrees, so a dev sub-agent works **in the main
repo** under a tight footprint. Delegate the bulky items (the guard, the test-ids split, the seams);
write the one-liners (`.gitattributes`, the workspace dir) inline. Force a StructuredOutput so the
return is machine-checkable, and route effort upward on the load-bearing ones:

```js
const SCHEMA = { type:'object', additionalProperties:false, properties:{
  item:        {type:'string'},                       // which Phase-0 item
  filesTouched:{type:'array', items:{type:'string'}}, // must match the item's intended footprint
  gateOutput:  {type:'string'},                        // the REAL typecheck/lint/test tail
  gatesPassed: {type:'boolean'},
  notes:       {type:'string'} },
  required:['item','filesTouched','gateOutput','gatesPassed','notes'] }
```

The sub-agent prompt carries: the item's footprint, the recipe below verbatim-in-spirit, "**touch
only your footprint; do not edit the coordination/state file**", and "run the gate in the repo and
return its real tail — do not summarize it." Effort: **xhigh** for the guard and the FK-ordered
seams (subtle, load-bearing); **high** for the test-ids split and the mechanical items. After it
returns, the orchestrator verifies the diff with real git (`git diff --stat`, `git status`), re-runs
the gate itself if the stakes warrant, then makes a small footprint-clear commit. **Never `git add
-A`** — stage exactly the item's files (the method's footprint-only discipline applies on main too).

---

## Migration-set guard

A structural test that catches a *bad merge* the unit tests pass green — a duplicate or out-of-order
migration from a careless integration corrupts only on a real device otherwise. It asserts, over the
whole migration set:

- versions are **strictly increasing, unique, and contiguous** (no gap, no repeat);
- **`DB_VERSION == count == max(version)`** — the declared schema version equals both the number of
  migrations and the highest version (catches a migration added without bumping `DB_VERSION`, or vice
  versa);
- **no duplicate `CREATE TABLE` / `ADD COLUMN`** across the set — scan the SQL of every migration and
  fail on a table created twice or a column added twice (the canonical bad-merge signature).

Write it as a real test in the project's runner (e.g. a jest test that imports the actual migration
array and `DB_VERSION`, not a hand-copied list — it must track the real source). **Generalize the
shape** to whatever registry the spec flags: any flat, ordered, merge-appended registry a bad union
can silently duplicate (a glyph map, a route table) deserves the same uniqueness/contiguity guard.

Do not stop at "the guard is written and green" — green proves nothing until you have seen it go RED
(next section). That is the whole point of A3.

---

## Prove the guard RED (the load-bearing beat — do not fake it)

The guard is worthless until you have watched it fail on a real fault. Every self-report on the first
full run matched real git, so no guard was *ever* exercised in real failure (A3); this beat is the
correction. Concretely:

1. **Isolate.** Inject the fault where it cannot reach main:
   - **preferred** — a throwaway worktree: `git worktree add <tmp> <main_branch>`, inject there, run,
     then tear down **junction-safe** (see § *Junction-safe teardown* — remove any `node_modules`
     junction *link* BEFORE `worktree remove`, or the removal recurses through the junction and eats
     the real `node_modules`);
   - **lightweight** — a **working-tree-only** edit in the main repo you will `git checkout -- .`
     before committing anything. Never commit the fault to main, and never leave it staged.
2. **Inject one real violation** — pick one and make it concrete: a **duplicate version** (two
   migrations at the same version), an **out-of-order** version, a **`DB_VERSION` / count mismatch**,
   or a **duplicate `CREATE TABLE`**. The fault must be the exact shape a bad merge produces.
3. **Run the guard and capture the REAL failing output** — the actual assertion message and the
   non-zero exit, pasted into the receipt, not paraphrased. If the guard stays green on the injected
   fault, **the guard is broken** — fix it and repeat until it fails on the fault. If a sub-agent did
   the injection, the orchestrator reads the captured failing output with its own eyes; a sentence
   that says "it went red" is not evidence (#1).
4. **Revert and prove clean** — restore the migration set, run the **full gate**, confirm green, and
   confirm `git status` is clean (the fault left no residue). Only now commit the guard.
5. **Record** the injected fault, the failing-output excerpt, and the green-after line in the receipt.
   S3's precondition is literally "guard proven RED" — this block is that proof.

> Optional hardening (lever 6 / S0's factored probe): inject a *control* fault that exercises the
> other guard-rails too (a lane out of footprint, an un-caught union) to prove they scream. Out of
> scope for the minimum Phase 0, but this is where it would live.

---

## Hoist cross-wave seams

For each edge the plan dissolves with a stub — **a dissolvable real edge or a neutralized hidden
edge; the playbook's seam list carries both**, one-to-one with the edges the DAG dropped — declare
the **interface now** — a **typed port + a no-op stub** — so the Wave-N consumer binds to a
contract on main and the producer fills the impl in a later wave. The critical path does not deepen,
and no lane invents its own divergent copy (a dissolved edge whose seam never ships forces exactly
that: the consumer's dev, finding no port on main, declares a local divergent one to go green).

The load-bearing parts are **position/ordering** and the **semantic contract**, not just the type.
Bake the consumer's constraint into the seam itself:

- **FK ordering (the one that bit — FN-7 / A1).** Under `foreign_keys=ON` with no `ON DELETE
  CASCADE`, a delete/purge of a referencing row MUST run **before** the referenced-row delete. If the
  seam is a delete-path stub, its position in the cascade — and a comment stating the order and *why*
  — is the whole value. A positionally-wrong seam is **worse than no seam**: the next dev follows the
  shipped code over the prose, so a mis-placed stub actively ships the FK bug. On the first full run
  this was caught *only* because the orchestrator read the real delete-path at the gate.
- **Anchor to the real consumer.** Read the actual consuming code (or its spec) and place the seam
  where that code will call it — do not infer the position from the seam's name. Leave a one-line
  comment: the ordering rule, and "S3's gate re-validates this seam against the real consumer."
- **Bake the semantic contract** (from the playbook's seam entry: units, rounding/precision, sign,
  empty/error behavior) into the port's doc comment, and commit the seam's **contract test,
  `skipped`** — it encodes the consumer's expectations against the real impl and is un-skipped by
  S3 at integration when the producer lane lands (it must go skipped→green, i.e. it would have
  been RED against the no-op). Consumers mock the port in their own tests; without this test,
  producer/consumer semantic drift across waves has no owner — centimes-vs-euros ships green.

Gate the seam (it must typecheck as a no-op) and commit it footprint-clear.

---

## `.gitattributes` merge policy

`merge=union` auto-merges concurrent appends — but only safely on a **genuinely flat** registry: a
sequence of simple one-line `export const X = …` blocks with **nothing structural after the last
one**. Set it precisely:

- **Union ONLY** the flat registries — after the test-ids split, each per-screen `test-ids/<screen>.ts`
  is flat and a good union candidate (each lane appends its own ids).
- **NEVER union the coordination/state file** a tool parses (the sprint/status file): union duplicates
  keys and breaks the parser.
- **NEVER union a structured registry or a contract `.test.ts`**: union splices conflicting hunks
  line-by-line and **silently produces invalid code** once the **3rd** concurrent appender lands (the
  first two often merge clean and lull you — FN-2/5/7). Keep `tsc` + the contract test as the backstop
  and resolve those by hand at integration.

The split (next section) is what *lets* you union safely — without it, union on the monolithic
`test-ids.ts` is exactly the corruption class lever 1 exists to kill.

**Prove the policy binds — the same A3 rule as the guard (a written policy is not a bound one):**

1. **`git check-attr merge <path>`** on **every** intended union file (each must print `union`) AND
   negative checks on the coordination/state file plus at least one structured registry / contract
   `.test.ts` (each must print *no* union). A pattern that silently no-ops (wrong path prefix —
   `test-ids/*.ts` when the files live at `src/test-ids/`) surfaces on every wave as raw 3-way
   conflicts on the hottest files; one that over-matches splices a structured file silently — and
   on a non-TS registry (JSON locale map, YAML route table) there is **no `tsc` backstop** to catch
   the splice at integration.
2. **One live merge trial**, once, in a throwaway worktree (junction-safe teardown, as the RED
   proof): two branches appending to a union'd file must auto-merge with both blocks intact; the
   same trial on a structured file must **not** union (a real conflict is the correct outcome).
3. **Record both checks** in the receipt's *Union policy proven* block and set
   `union_policy_proven: true` in its frontmatter — S3 keys off it like `guard_proven_red`.

---

## Pre-provision native deps

List every native dep the backlog needs (from the architecture/epics, via the spec) and install them
**once on main** before the waves — this spares the per-lane declare-in-package.json + `.d.ts` shim +
install-at-integration dance (cost C3), which can't `npm install` through the shared `node_modules`
junction anyway. The trap is the install, not the listing — **verify before you trust**:

- **Full peer/transitive set** — a v14 test lib needed a separate `test-renderer` peer that wasn't
  pulled automatically (FN-4). Install with the project's tool (`npx expo install` on Expo, so RN
  versions stay aligned) and read what it actually added.
- **Real export surface** — confirm the installed package's actual API matches what a lane will
  assume. An fs API had moved to a `/legacy` subpath between the assumed and the real version (FN-1);
  the lane that shimmed the old surface would have broken at integration. Open the installed package,
  confirm the import path and the exported names empirically, and note them in the receipt so the
  lanes bind to the real surface.

Gate after install (a new dep can shift the typecheck/lint surface) and commit the lockfile +
`package.json` together.

---

## Split test-ids per screen + barrel

The single serious technical fix (lever 1). Replace the monolithic `test-ids.ts` with one file per
screen plus a barrel:

- `test-ids/<screen>.ts` — each screen's ids, flat (`export const FOO = 'foo'`), so a lane writing a
  screen touches **only its own file**: zero collision, zero union splice, zero manual resolution at
  integration. This is what dissolves cost C1 — the recurring, every-wave hand-resolution pain.
- `test-ids/index.ts` (the barrel) — re-exports each screen file so existing `import { … } from
  '.../test-ids'` call sites keep working. The barrel itself is append-only (one `export *` line per
  screen); list it in the playbook so lanes append their screen's line in their own commit, or
  pre-create all the lines in Phase 0 if the screen set is known.
- **Migrate existing usages** to the split (or keep the barrel API identical so nothing else changes).
  Gate: `tsc` must pass with the ids resolving through the barrel, and the test-ids contract test (if
  any) must still find every id.

List the per-screen split the spec implies in the receipt — it is the durable record of which screen
owns which file when the lanes start appending.

---

## Single-source shared helpers (from the plan's duplication clusters)

For each computation the playbook lists under *Shared helpers to single-source* with a Phase-0
home: implement the small **pure** helper once on main — a **real impl with its unit test**, not a
no-op stub (a stub of a *formula* forces every consumer to mock it, which recreates the divergence
it exists to prevent) — so every consumer card's "import it, never re-implement" constraint has a
real target on main before any lane starts. Where the playbook instead designated a producing
story, build **nothing** here — the constraint rides the cards and the wave order. Gate and commit
footprint-clear.

## Shared test fixtures/builders (optional — the test-ids move, applied to the test layer)

When the playbook lists entities ≥2 lanes will seed: pre-provision **one fixture-builder home** on
main, referenced by every consumer card. N hand-rolled seeders with subtly different assumptions
(VAT-inclusive vs VAT-exclusive amounts) each stay green against their own fixture and contradict
each other at the first parity test — at which point the failure cannot be attributed (helper
divergence vs fixture divergence) without archaeology.

---

## Worktree workspace + ownership policy

Two small, durable acts:

- **Workspace** — ensure the sibling `worktree_workspace` directory exists (the default is
  `{project-root}/../<project>-wt`). The lanes' durable worktrees live here; create it now so Wave 1
  doesn't race on it. Do not put it inside the repo.
- **Ownership policy** — record, in the receipt and wherever the orchestrator will read it, the rule
  the waves obey without exception: **lanes NEVER write the coordination/state file (sprint/status);
  the orchestrator owns it 100%** and updates main at integration to reflect merged reality. This is a
  mechanical consequence of emulate-vs-invoke (an emulating lane simply never touches it) and the
  reason the state file must never be unioned.

---

## Junction-safe teardown (method §8 — reuse it for any throwaway worktree)

Whenever you stand up a throwaway worktree (the RED proof) and it borrowed `node_modules` via a
junction/symlink from the main repo:

1. Remove the **junction LINK only** first — `cmd /c rmdir "<wt>/node_modules"` or
   `[System.IO.Directory]::Delete(path,false)`. Removing the link, not its target.
2. **Then** `git worktree remove <wt>` (or `--force` if needed).
3. **Verify the main repo's `node_modules` is intact** — if you `worktree remove` before dropping the
   junction, the removal can recurse through the link and delete your real `node_modules`. Check the
   count is unchanged.

Never install through a junction. And never rewrite pre-session history during cleanup — use
forward-only fixes (`git revert`, a new corrective commit); some harness classifiers block
`--amend`/`reset` on commits that pre-date the session (method §8).
