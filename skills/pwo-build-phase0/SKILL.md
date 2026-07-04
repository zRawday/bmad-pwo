---
name: pwo-build-phase0
description: "STEP 4 of 6. Implement and PROVE the Phase-0 guard-rails on the main branch before the first wave (migration-set guard proven RED, seams, .gitattributes, pre-provisioned deps, test-ids split per screen). Use when the user says \"build phase 0\", \"ship the phase-0 infra\", \"pwo phase 0\", \"PWO S2\", or after the wave plan is validated, before the first wave."
---

# pwo-build-phase0

Implement on the **main branch** — and **prove** — the safety infrastructure the wave plan
decided, so every wave that follows runs on solid ground. You consume the **playbook**
(`pwo-plan-waves`, S1) and build exactly its **Phase 0 spec**: a structural **migration-set
guard** proven to fail on a bad merge, the **hoisted cross-wave seams** (each carrying its
consumer's ordering constraint), the **`.gitattributes`** merge policy, the **pre-provisioned
native deps**, the **test-ids split per screen**, and the **worktree workspace + state-file
ownership** policy. You are the bookend before the waves: when you finish, main is green, the
guard is **proven RED**, and `pwo-run-wave` (S3) keys off your receipt. This is *code to merge
and prove*, not analysis — distinct from S1.

**The two non-negotiables:**

1. **Prove the migration-set guard goes RED by fault injection.** A guard that has never failed
   is not a guard — it is an assumption (blind spot A3: every self-report matched real git, so
   no guard was ever tested in real failure). Inject a real violation (a duplicate / out-of-order
   migration), run the guard, **capture the real failing output with your own eyes**, then fully
   revert and confirm green. Never trust a sub-agent's *claim* that it went red — verify the RED
   from real command output (invariant #1).
2. **Split `test-ids` into one file per screen + a barrel.** This is **lever 1** — the single
   serious technical fix. It closes the entire union-corruption class (cost C1) that recurred on
   *every* wave: when each lane writes only its own `test-ids/<screen>.ts`, there is zero
   collision, zero `merge=union`, zero manual resolution at integration.

And the suite's standing priority — **main green > speed > cost** (route effort *upward* on
anything load-bearing; token economy is not a goal) — and its habit of **verifying from the source
of truth, never a self-report**.

## Resolution rules

- Bare paths and `{skill-root}` (e.g. `references/phase0-build.md`) resolve from this skill's installed directory.
- `{project-root}` → the project working directory.
- `pwo-build-phase0` → the skill directory's basename.

## On Activation

Human-triggered, single-pass on the main branch. There is no runtime memlog and no resume store:
your durable state **is git** (the commits you land) plus the **phase-0-receipt** artifact you
write as you go — to resume an interrupted run, re-read the receipt and reconcile it against real
git, never a remembered state (#1). Talk to the human in `{communication_language}`.

1. **Config.** Read `{project-root}/_bmad/config.yaml` (and `.user.yaml` if present), defaults for
   anything missing. Resolve `{project-name}` (else the repo folder name), `{output_folder}`
   (else `{project-root}/_bmad-output`), `{communication_language}`, `main_branch` (else
   `git rev-parse --abbrev-ref HEAD`), and `worktree_workspace`.
2. **Inputs:**
   - **playbook** — `{output_folder}/pwo/playbook.md` (S1's output). The **`Phase 0 spec (for
     S2)`** section is your authoritative work list. If it is absent, **stop and escalate** — S2
     has nothing to build without it.
   - **receipt path** — default `{output_folder}/pwo/phase-0-receipt.md`.

## Confirm the ground before you touch main

Read the playbook's `Phase 0 spec (for S2)` section in full — it lists each item *concretely*
(which guard, which seams + their ordering constraints, which deps, which per-screen test-id
split). Then establish the starting truth from **real git**: capture `main_branch`'s HEAD, confirm
the tree is clean (or note pre-existing untracked files to ignore), and confirm the gate chain
(`typecheck → lint → test`) is **green before you start** — you cannot prove you kept main green if
it was already red. Record the baseline HEAD in the receipt. An item the spec marks absent (no
SQLite migrations, no test-ids, no native deps) is **skipped, not faked** — record it `n/a` with
the reason; do not invent infra the project does not need.

## Build each Phase-0 item, gate, and verify from real git

Work through the spec's items. `references/phase0-build.md` carries the concrete recipe for each
— read the entry for the item you are building. **Delegate the bulky coding** to a dev sub-agent
that returns a StructuredOutput (what it built, files touched, gate result), routing effort
*upward* on the load-bearing ones (the guard, the FK-ordered seams → high/xhigh; the mechanical
workspace/`.gitattributes` → high); or write the smaller items inline. If dev sub-agents are
unavailable, the orchestrator builds the bulky items inline too — only throughput changes, never
the discipline. Whoever codes, **the orchestrator keeps the judgement**: after each item, verify
the diff against real git (footprint only, no stray coordination-file edit), run the gate, and
commit small and footprint-clear on main (or a short-lived branch ff-merged to it). The items,
with their load-bearing edge (recipes in `references/phase0-build.md`):

- **Migration-set guard (+ prove RED).** A structural test over the whole migration set
  (strictly-increasing/unique/contiguous versions, `DB_VERSION == count == max`, no duplicate
  `CREATE TABLE`/`ADD COLUMN`); generalize the shape to any registry the spec flags. Proving it
  RED is its own beat below — the load-bearing, easy-to-fake step.
- **Hoist cross-wave seams.** A typed port + no-op stub per dissolved edge — **dissolvable real
  edges and neutralized hidden edges alike** (the playbook lists both, one-to-one) — with the
  consumer's **position/ordering constraint baked in** — above all **FK ordering**: a
  positionally-wrong seam is *worse* than none, because the dev follows shipped code over prose
  (FN-7/A1). Anchor it to the real consumer, verify its position against the actual delete-path,
  and commit its **contract test, skipped** (units/rounding/sign/empty-error — S3 un-skips it when
  the producer lands; drift across waves otherwise has no owner).
- **Single-source the shared helpers** the playbook lists (real pure impls + unit tests, not
  stubs), and **pre-provision the shared test fixtures/builders** if listed — the plan's
  duplication clusters, killed at the source (recipes in the reference).
- **`.gitattributes` merge policy — written AND proven to bind.** `merge=union` **only** on
  genuinely flat registries (the split test-id files qualify); **never** on the coordination/state
  file or a structured registry / contract `.test.ts` — union splices those silently. Then prove
  it: `git check-attr` on every union'd path and negatively on the state file + one structured
  file, plus one live concurrent-append merge trial (recipe in the reference) — a policy whose
  pattern silently no-ops or over-matches is a corruption class with no `tsc` backstop on non-TS
  registries. Record `union_policy_proven` in the receipt; S3 keys off it.
- **Pre-provision native deps.** Install every native dep the backlog needs once on main (sparing
  the per-lane shim dance, C3) — but **verify the full peer/transitive set and the real installed
  export surface first** (FN-4/FN-1 both bit at integration); bind to the actual API, not the
  assumed one.
- **Split test-ids per screen + barrel.** `test-ids/<screen>.ts` per screen + a barrel
  re-exporting them — non-negotiable #2 (lever 1: each lane writes only its own file, nothing to
  union, nothing to hand-resolve).
- **Worktree workspace + ownership policy.** Ensure the sibling `worktree_workspace` exists, and
  record the rule the waves obey: **lanes never write the coordination/state file; the
  orchestrator owns it 100%.**

After every commit, re-gate and re-check real git — two green items can combine red, and the whole
point of Phase 0 is that main never goes red on the way in.

## Prove the migration-set guard RED (the load-bearing beat)

The guard is worthless until you have watched it fail — and this is the step that is trivial to
fake, the reason A3 exists. Inject a real bad-merge fault **in isolation** (a throwaway worktree or
a working-tree-only edit — **never a commit to main**), **run the guard and read its real failing
output with your own eyes** (a sub-agent's *claim* of red does not count — verify it from the real
output), then revert and confirm the set is clean and the gate green again *before* committing the
guard. **Record in the receipt** the fault injected, the real failing-output excerpt, and the
green-after — S3 reads it as the proof the guard is real. The concrete recipe (isolation, the fault
shapes, junction-safe teardown) is in `references/phase0-build.md` § *Prove the guard RED*.

## Assemble the receipt and bring it to the human

Write the artifact from `assets/phase-0-receipt-template.md` to the resolved receipt path: the
baseline + final main HEAD, a per-item table (done / n-a · artifact path · commit · gate result),
the **captured RED proof** block, the deps' verified export-surface notes, the test-id split list,
and the ownership policy. Then **bring it to the human for the one validation that gates the waves**
(AskUserQuestion): *is Phase 0 merged on main, green, the guard proven RED, and the union policy
proven to bind (or n-a)?* Show the real-git evidence (the HEAD, the gate output, the RED excerpt,
the check-attr/merge-trial output), not a summary you are asking them to trust.
Record the sign-off in the receipt. **If the human does not validate** — the RED proof
unconvincing, a footprint looked dirty, a gate not actually green — do **not** write the validated
hand-off and do **not** let S3 key off the receipt: return to the failing item, re-prove RED or
re-gate, and re-present the real-git evidence (the same state the headless `pending-human` /
`blocked` path holds). Only a validated receipt is the hand-off to `pwo-run-wave` (S3).

## Hand off to the next step

PWO is a **guided pipeline**: each step ends by handing the user a paste-ready prompt for the next one,
run in a **fresh session**. **Only once the human has validated the receipt** (Phase 0 merged on main,
green, guard **proven RED**, union policy **proven to bind** or `n-a`) do you hand off — if validation fails, you do **not** emit a handoff (return
to the failing item, as above). When validated, emit — in **English** — a single fenced code block whose
**first token is the next command** followed by a self-contained prompt that names **this wave's lanes +
their ⚙ anti-collision constraints (verbatim from the playbook's cards)**, then tell the user: **"Copy
this into a NEW Claude Code session (fresh context) to run it."** Resolve every `{token}` to its real
value (the final main `{sha}`, the receipt/playbook paths, the project name, the Wave 1 lanes). Skip in
headless mode (the JSON tail carries `mainHead` + `guardProvenRed` + `unionPolicyProven` + `receipt`).

```
/pwo-run-wave Phase 0 is merged and green on {main_branch} @ {sha} for {project-name}, guard proven RED,
union policy {proven to bind | n-a} (receipt: {output_folder}/pwo/phase-0-receipt.md, overall: ready). Run Wave 1 end-to-end keeping main
green, in THIS fresh session. This wave's lanes + their ⚙ anti-collision constraints (VERBATIM) are in
the playbook ({output_folder}/pwo/playbook.md): {list each lane: key (tier) · ⚙ "constraint"}. Pipeline:
WF1 create (emulate) → GATE → WF2 dev (emulate; dev=xhigh, critical=max) → verify real git → code-review
TOP-LEVEL (effort max) → serial integration → end-of-wave critic → smoke → Field Note + next-wave
handoff. Read the playbook + Phase 0 receipt first, propose the dispatch plan, and get my GO before WF1.
```

## Headless output

When invoked headless, skip the interactive validation: still build every item, write the receipt,
mark the human sign-off `pending-human` (with the real-git evidence attached), and return JSON only —
`{ "status": "complete" | "blocked", "mainHead": "<sha>", "guardProvenRed": true | false, "unionPolicyProven": true | false | "n-a", "itemsDone": ["migration-guard", "test-ids-split", …], "itemsSkipped": ["…"], "receipt": "<path>", "pendingHuman": ["phase-0-validation"] }`
(`"blocked"` with a one-line `reason` if the playbook's Phase 0 spec is absent, if main was red
before you started, or if the guard could not be made to go RED — that proof is never waived).
