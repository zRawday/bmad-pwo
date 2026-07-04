# Wave Pipeline — the orchestrator's recipe for one wave

The per-step recipe SKILL.md routes here for: the **model × effort table**, the **WF1/WF2 script
skeletons + StructuredOutput schemas** (with the emulate / leak-guard / footprint-only parade),
the **gate fact-extraction recipe**, the **§7 real-git verification checklist**, the **serial
integration + junction-safe teardown** order, the **end-of-wave critic** checklist, the **S5
smoke call** wiring, and the **Smoke triage** protocol (the FAIL/BLOCKED path). Read the section
for the step you are running. This file stands alone — it does not assume SKILL.md is still in
context.

`REPO` = the absolute main-repo path · `WT` = the absolute `worktree_workspace` · `MAIN` =
`main_branch` · `IMPL` = the project-root-**relative** implementation-artifacts dir (bmm's
`implementation_artifacts`, resolved from `{project-root}/_bmad/config.yaml`, fallback
`_bmad-output/implementation` — it must be the **same** directory bmm's create/dev-story scan, or
the specs are invisible to every bmm tool and to later waves' example lookup). `LANES` comes from
this wave's per-lane cards in the playbook (`pwo-plan-waves`): each card carries `key`, `title`,
`footprint`, `hotspots`, the **⚙ constraint verbatim**, and a `tier`
(`trivial | standard | critical`). `PATTERNS` = the ≤10-line patterns digest from the previous
wave's run-record (established idioms to follow / mistakes not to repeat, distilled from the run
log's Field Notes + prior gate amendments + prior review corrections; empty on wave 1) —
**precedence: the card's ⚙ constraint and any gate amendment OVERRIDE the digest where they
conflict.**

---

## The model × effort table (read while constructing every `agent()` call)

The orchestrator chooses **both the model and the effort of every delegation** — sub-agents never
inherit the session's model by default. The routing rule: **the session's top-tier model (the one
running YOU) is reserved for the orchestrator's own load-bearing judgement** — the GATE, the fold,
the hand-resolutions; delegated work runs on the strongest *coding* tier (`model:'opus'`) and
routes **down** (`model:'sonnet'`) where the step is observation/protocol rather than reasoning —
never up. A sub-agent runs the top tier **only on explicit user request** (record it in the
run-record). If a named tier is unavailable in the harness, fall back to the nearest tier **at or
below** it — never silently upgrade. Within the routed tier, route effort **upward** — the cap
goes on whatever *judges* or *codes at stakes*; drop back only on the purely procedural (main
green > speed > cost).

| Task | Model | Effort | Why |
| ---- | :---: | :----: | --- |
| create (emulate) | `opus` | **high** | the gate catches content, but a poor spec burns the gate's scarce budget |
| dev (emulate) | `opus` | **xhigh** | single pass, no retry net → strongest coding tier first try; **`max` on a `critical` lane** (tiering upward is the only safe direction) |
| code-review / delta re-review | `opus` | **max** | the last adversarial net; it fights the dev's confirmation bias |
| patch (load-bearing) / triage hotfix | `opus` | **xhigh** / **high** | subtle coding (rounding, FK, boundary); a hotfix is scoped, its review runs `opus`·max |
| duplication-scout / merged-result sweep / S5 smoke | `sonnet` | **high** | bounded by observation fidelity + protocol, not reasoning depth — don't overthink the procedural |

The lane's `tier` (from S1's card) does two concrete jobs at runtime: it **arms the anti-mis-tiering
guard at the gate** (a `critical` lane whose spec reads thin gets sent back) and it **bumps
`critical` lanes to `dev=opus·max`**.

---

## WF1 — create (parallel, EMULATE)

One workflow agent per lane. The load-bearing steps are **emulate-not-invoke**, **absolute
worktree paths**, the **survey-before-you-specify** (main already holds implementations and
behaviors the card, frozen at S1, could not know about), an **explicit spec-only commit**, and a
**report-only main-repo leak-guard**. Cut every lane's worktree from the **same** main snapshot
(commit nothing to main between lane creations — asymmetric bases are harmless but the
integration rebase has to fix them). Before dispatch, capture the pre-wave baseline
`git -C REPO status --porcelain` into the run-record — it is the "known pre-existing" list every
lane's leak-guard compares against.

```js
export const meta = { name: 'waveN-create', description: 'emulate create-story per lane', phases: [{ title: 'Create' }] }
const REPO = '<abs main repo>', WT = '<abs worktree_workspace>', MAIN = '<main_branch>', IMPL = '<impl-artifacts dir, repo-relative>'
const BASELINE_STATUS = `<the pre-wave 'git -C REPO status --porcelain' output, captured by the orchestrator>`
const PATTERNS = `<the ≤10-line patterns digest from the previous run-record; empty on wave 1>`
const LANES = [ /* { key, slug, titleHint, constraint:'<the card ⚙ constraint, verbatim>' }, … */ ]
const SCHEMA = { type:'object', additionalProperties:false, properties:{
  storyKey:{type:'string'}, branch:{type:'string'}, specPath:{type:'string'}, specCommitSha:{type:'string'},
  specOnlyCommit:{type:'boolean'}, mainRepoClean:{type:'boolean'}, acHeadlines:{type:'string'},
  filesTouched:{type:'string'}, notes:{type:'string'} },
  required:['storyKey','branch','specPath','specCommitSha','specOnlyCommit','mainRepoClean','acHeadlines','filesTouched','notes'] }
function buildPrompt(l){ const wt = `${WT}/${l.key}`; return `Autonomous create lane for Story ${l.key} (${l.titleHint}).
0) BASE = git -C "${REPO}" rev-parse HEAD (the main branch; leave it UNTOUCHED).
1) git -C "${REPO}" worktree add "${wt}" -b story/${l.key} ${MAIN}  (retry 2-3x on a .git lock; reuse an existing
   worktree ONLY if it is clean AND cut from the current ${MAIN} HEAD — anything stale: remove it and re-add).
2) SURVEY BEFORE YOU SPECIFY (reads from ${REPO} are fine; WRITE only under ${wt}):
   a) grep ${REPO} for existing implementations of every computation/predicate the story needs — the spec IMPORTS an
      existing single-source helper (list each in a "Consumes from main" section) and may re-specify a computation
      ONLY after recording that no implementation exists on main.
   b) for EVERY existing file in your footprint (UPDATE, not NEW): read its full current body and record a Dev Notes
      block — current behavior, what this story changes, what MUST be preserved. A behavior required for the system
      to keep working is a requirement even if no AC states it.
3) EMULATE create-story — do NOT invoke the Skill tool (it writes to the MAIN repo and leaks). Use the installed bmm
   template as the FORMAT (${REPO}/.claude/skills/bmad-create-story/template.md — locate it under the installed
   bmad-create-story skill if the path differs; from wave 2 on, a spec a prior wave merged is an acceptable format
   reference; if neither exists, mirror these sections): Story / ACs / Tasks-Subtasks MAPPED to ACs / Dev Notes / File List.
   AUTHOR THE SPEC DIRECTLY to ${wt}/${IMPL}/${l.slug}.md, baking this constraint VERBATIM: "${l.constraint}".
   Patterns digest (the ⚙ constraint and any gate amendment OVERRIDE it where they conflict): ${PATTERNS}
4) Commit ONLY the spec (never git add -A): git -C "${wt}" add "${IMPL}/${l.slug}.md"
   && git -C "${wt}" commit -m "docs(story ${l.key}): create story spec".
5) LEAK-GUARD — OBSERVE AND REPORT ONLY; you NEVER write, clean, checkout or delete anything under ${REPO}:
   assert git -C "${REPO}" rev-parse HEAD == BASE, and git -C "${REPO}" status --porcelain matches this pre-wave
   baseline: ${BASELINE_STATUS}. On ANY delta: report mainRepoClean:false + the raw porcelain excerpt in notes and
   stop — the orchestrator (the sole writer) triages serially; another lane's leak or the user's WIP is NOT yours to fix.
Return the schema; filesTouched MUST be exactly the one spec file; acHeadlines = the spec's AC one-liners.` }
phase('Create')
return { specs: (await parallel(LANES.map(l => () =>
  agent(buildPrompt(l), { label:`create:${l.key}`, phase:'Create', schema:SCHEMA, model:'opus', effort:'high' })))).filter(Boolean) }
```

---

## GATE — the fact-extraction pre-pass (you; not a sub-agent)

This is the orchestrator's irreplaceable judgement and the one step that has repeatedly saved the
build. It is light (≈20 lines of real code per seam) — keep it in your own context; do **not**
delegate it.

1. **Extract facts from the real code, not the prose.** For every seam / function / type the lane
   *consumes*, open its **real body** and read it. A spec's prose ("km × barème", "purge here") is
   the earlier author's *assumption*; the shipped code is ground truth, and the dev follows shipped
   code over prose. Two things this catches that nothing downstream can:
   - **A positionally-wrong forward-seam.** A reserved comment/stub left by an earlier wave encodes
     *that* wave's ordering. Re-validate its **position** against this consumer's NEW constraints —
     above all **FK ordering**: under `PRAGMA foreign_keys = ON` with no cascade, a purge/child
     delete must run **before** the referenced-row delete, or it FK-throws on-device. A
     positionally-wrong seam is *more* dangerous than no seam — it actively misleads. The FK-blind
     in-memory double cannot catch a wrong order, so the later review must **prove order**, not just
     zero-orphan.
   - **A prose-vs-type mismatch.** When the constraint prose and the canonical in-repo type diverge,
     the spec must point at the type ("follow `<type>`; the model wins where this prose differs").
2. **Reconcile the cards with merged reality.** The playbook froze before any code existed: verify
   this wave's cards — footprint paths, consumed symbols, the ⚙ constraint's anchor type, entry
   points — against current main. A stale card is amended **in the playbook** (logged like a gate
   amendment, human-visible), never silently obeyed or silently ignored. A discovered missing/wrong
   edge **defers the affected lane** (never improvise around the DAG); if it invalidates the
   remaining cut, escalate to S1's **scoped re-plan**.
3. **Verify the spec is dev-ready:** the ⚙ constraint is baked verbatim; the footprint matches the
   card; the ACs match the epic; the template sections are present (**Tasks mapped to ACs** — a
   Tasks-less spec silently degrades WF2 to "implement from prose"); the **"Consumes from main"
   list is verified against the real exports on main** (a spec that re-specifies a helper main
   already has goes back, like a thin critical spec); a **preserved-behaviors Dev Notes block
   exists for every UPDATE file in the footprint** (a spec without one on a non-NEW footprint goes
   back); and the spec does not contradict the patterns digest.
4. **Cross-check every load-bearing value** (seeds, barèmes, fiscal constants) against the source of
   truth yourself — never let a `critical` lane carry an unverified number.
5. **Anti-mis-tiering guard:** a lane the card tagged `critical` whose spec reads thin is under-spec'd
   — send it back (and it runs `dev=max`).
6. **Amend in-worktree, surgically.** Where the gate resolves something — a thin spec, a moved seam,
   or a **product/fiscal decision taken with the human** — inject a prominent block at the **top** of
   the in-worktree spec and fix only the 1–2 most-contradicting lines (don't rewrite it):

   ```
   ⚙ ORCHESTRATOR GATE AMENDMENT — OVERRIDES CONFLICTING PROSE BELOW
   <the authoritative instruction: e.g. "purge pending_distance BEFORE the étape delete, in the
    étape-referencing block"; or "defer train/autre dispatch; attribute manual notes by annee(createdAt)">
   ```

   Commit the amendment **spec-only** on the lane branch **before** WF2. The high-effort dev agents
   honor a top-of-spec amendment cleanly — have each report `amendmentHonored`.

---

## WF2 — dev (parallel, EMULATE; the worktree already holds the gated spec)

Same parade as WF1. Set each lane's effort from the table: **xhigh**, **max** if its card tier is
`critical`. **Robustness:** on a red gate, the agent retries a bounded number of times under
footprint before returning red; if a lane process dies, **`resumeFromRunId`** the workflow rather
than restarting — it protects the xhigh/max investment in the lanes that already finished.

```js
const BASELINE_STATUS = `<the same pre-wave 'status --porcelain' baseline as WF1>`
const PATTERNS = `<the same ≤10-line patterns digest as WF1>`
const SCHEMA = { type:'object', additionalProperties:false, properties:{
  storyKey:{type:'string'}, branch:{type:'string'}, gatesPassed:{type:'boolean'}, testCount:{type:'number'},
  newTestCount:{type:'number'}, diffStat:{type:'string'}, featCommitSha:{type:'string'}, footprintOnly:{type:'boolean'},
  mainRepoClean:{type:'boolean'}, amendmentHonored:{type:'boolean'}, autocommitHandling:{type:'string'}, notes:{type:'string'} },
  required:['storyKey','branch','gatesPassed','testCount','newTestCount','diffStat','featCommitSha','footprintOnly','mainRepoClean','amendmentHonored','autocommitHandling','notes'] }
function buildPrompt(l){ const wt = `${WT}/${l.key}`; return `Autonomous dev lane for Story ${l.key}. Worktree ${wt} on story/${l.key} holds the gated spec.
0) BASE = git -C "${REPO}" rev-parse HEAD (leave main UNTOUCHED). cd "${wt}" so cwd-relative commands stay local.
1) Junction node_modules: link ${wt}/node_modules → ${REPO}/node_modules. NEVER install through the junction.
2) EMULATE dev-story: implement DIRECTLY from the spec's Tasks/ACs (honor the top-of-spec ⚙ GATE AMENDMENT if present,
   the "Consumes from main" imports, the preserved-behaviors Dev Notes, and any named in-repo pattern) to ABSOLUTE
   ${wt}/ paths. Per Task/AC: write the FAILING test first, run it, confirm it fails, then implement to green
   (red→green per AC — a test that never failed discriminates nothing). Implement NOTHING not mapped to a spec Task.
   Honor every footprint / hotspot / purity / deferred-import rule. Do NOT invoke the Skill tool.
   Patterns digest (the ⚙ constraint and the gate amendment OVERRIDE it): ${PATTERNS}
3) Gates IN THE WORKTREE: typecheck && lint && test (all against ${wt}); fix under footprint until green — but NEVER
   by weakening or deleting a failing assertion, and NEVER by re-implementing a symbol your spec says a parent or
   main produces: if green requires either, or anything outside your footprint, STOP and return gatesPassed:false
   with the reason in notes — the DAG may be wrong, and that is the orchestrator's call, not yours.
   Capture testCount (suite total) AND newTestCount (tests THIS lane added).
4) COMMIT explicitly, footprint-only (NEVER git add -A): git -C "${wt}" add <footprint files>
   && commit -m "feat(story ${l.key}): <subject>". If a stray autocommit ever fires, normalize to one footprint-only
   feat commit (amend out any coordination-file/stray paths).
5) LEAK-GUARD (as WF1 — OBSERVE AND REPORT ONLY, never write/clean/delete under ${REPO}): main HEAD == BASE, and
   git -C "${REPO}" status --porcelain matches this pre-wave baseline: ${BASELINE_STATUS}. On any delta report
   mainRepoClean:false + the raw excerpt in notes.
Return the schema (diffStat = git diff --stat <specCommit> HEAD).` }
phase('Dev')
return { devs: (await parallel(LANES.map(l => () =>
  agent(buildPrompt(l), { label:`dev:${l.key}`, phase:'Dev', schema:SCHEMA, model:'opus', effort: l.tier === 'critical' ? 'max' : 'xhigh' })))).filter(Boolean) }
```

---

## §7 — verify each lane with REAL git (before any merge)

Workflow agents **self-report**; re-check from the source of truth. Across a full build the
self-reports matched real git 100% — and this re-check stays non-negotiable, because it is how you
catch a parallelization defect the gates pass green. Per lane:

- **Footprint compliance** — `git -C <wt> diff --stat <specCommit> HEAD` touches **only** the card's
  files. A stray file = scope creep or a mis-routed edit; investigate before merge.
- **Spec-commit audit** — `git -C <wt> show --stat <specCommit>` lists **exactly the one spec file**
  (the `<specCommit>..HEAD` range structurally excludes that commit — re-derive `specOnlyCommit`
  from git, never trust the WF1 boolean), and `git -C <wt> log --oneline <MAIN>..HEAD -- <state-file
  path>` is **empty** — no lane commit ever touches the coordination file, and this is where a
  violation would hide.
- **Test reality** — the lane's diff adds real tests (`newTestCount` is a self-report; the diff is
  the truth). A `standard`/`critical` lane whose diff touches zero test files goes to the reviewer
  **flagged**.
- **Actual disjointness** — recompute the pairwise intersection over the lanes' REAL `diff --stat`
  file lists (S1's check ran on predictions; this is ground truth). Any file two lanes actually
  touched that is not a declared hotspot **stops the integration order** until resolved.
- **Hotspot discipline** — exactly one new migration object at its **pre-allocated** version (the
  playbook's allocation table names it — verify the number, not just the count); append-only
  labelled blocks on registries/barrels; a non-owner of a co-composed screen edits only
  its own child component (one slot line) and preserves the root testID.
- **Registry-merge integrity** — see Serial integration: re-gate after every rebase; a "clean" union
  on a structured file is not to be trusted.
- **Producer→consumer sanity** — the lane consumes only capabilities already on main. A red
  post-rebase gate usually means a parent was not actually merged.
- **Cross-check the real tree** — `git worktree list`, `git log --oneline`, `git show --stat <feat>`,
  and the main branch `rev-parse` / `status` (still == BASE, still clean).

---

## Code-review — TOP-LEVEL (never nested; Fact 1)

The review skill fans out into parallel layers, so it **cannot** run inside a workflow — run it
top-level: hand the human one prompt + worktree path per lane for **parallel terminals**, or run
them yourself as **Agent calls** in the main loop (pass `model:'opus'` — the routing table
applies to Agent calls too). **Opus-class, effort max.** Give each reviewer the **diff
range**, the **spec** (including its preserved-behaviors Dev Notes — a removed behavior no test
covered is a finding, not a cleanup), the four lenses (correctness / edge-cases / acceptance /
**coherence**), the lane's invariants, and **authority to apply safe patches in the worktree +
re-gate** (a load-bearing patch is opus·xhigh).

**Neutralize the review skill's sequential-flow write-backs** — the same emulate-don't-invoke
logic, applied to the review's side-effects. Bake into every reviewer prompt: the reviewer
**writes NOTHING under the main repo** (no `deferred-work` append, no sprint-status touch, no
story-tracking update — N parallel reviewers racing one main-repo file corrupts the very ledger
S4 seeds from); defer-findings come back **in the StructuredOutput** and the orchestrator folds
them into the ledger serially at integration; every applied patch AND any Review-Findings note is
**committed on the lane branch** before the reviewer returns (an uncommitted worktree edit dies at
the rebase); and **no lane enters serial integration with an open patch/decision-needed finding**
— the orchestrator resolves each (apply + re-gate · human decision · explicit defer with reason)
before that lane's rebase.

The prompts that earn the review its keep:

- **Sweep, don't spot-check.** For any money math or live-derived UI figure, sweep the **realistic
  input range** against an independently re-derived expectation (a green unit suite probing one
  magnitude misses a half-cent rounding bug or an out-of-band clamp).
- **"Is the policy tested, or mocked away?"** For any mandatory side-effecting policy a test commonly
  stubs (throttle / retry / rate-limit / auth / debounce), verify the policy **itself** is exercised
  with the stub **off** — a 100%-green suite that disables it everywhere is a silent coverage hole the
  test count hides.
- **The FK blind-spot.** When a lane adds the first rows referencing an existing table via a
  non-cascading FK, the review must **prove the delete order**, not just zero-orphan — the in-memory
  double enforces no FKs and surfaces a wrong order only as an orphan, never as the on-device throw.
- **Coherence (the fourth lens).** Does the diff follow the project-context conventions and the
  nearest in-repo exemplar — error handling, state management, naming, module layout? A deviation
  is a finding, patchable like any other. Per-lane review can only see one diff; the cross-lane
  version of this check belongs to the end-of-wave critic.

---

## Serial integration (you; incompressible, strictly ordered)

Two green branches can combine red, so this phase is serial and re-gated. Integrate per lane in
**DAG order, then ascending schema version**. File-truly-disjoint lanes (engine folders, child
components) may be rebased as a group and gated once.

1. **Sync + rebase.** Sync main; **rebase** the lane onto current main (lanes cut from an earlier
   snapshot **diverge** once you land any orchestrator commit, so a bare `--ff-only` fails — rebase
   first; disjoint lanes replay trivially).
2. **Resolve registry/test-id/glyph hotspots BY HAND.** Phase 0's per-screen test-id split should
   have killed most of this. Where a structured registry still collides, the fix is **deterministic**
   — keep **every** lane's block, in order. The recurring shapes:
   - **`test-ids.ts` (union):** union **silently** nests the new block inside the prior one, dropping
     its `} as const;` — even when git reports the rebase **conflict-free**. Only `tsc` (TS1005 /
     TS1435) catches it. Fix: **re-close the spliced block** (insert `} as const;` before the spliced
     comment); the **first** appender onto an untouched file applies clean.
   - **The contract `.test.ts` (union reverted):** sometimes auto-merges clean, sometimes a normal
     3-way — **inspect, don't assume**. When it conflicts, the previously-last `it()` block is
     *partial* (it shares the file's trailing `});});`), so **re-insert the partial close** at the
     `=======` seam **before** stripping the `<<<<<<<` / `>>>>>>>` markers.
   - **A shared glyph/icon map or a multi-lane store:** classify each hunk — `keep-both` for
     interface/impl blocks, but **keep-HEAD** for a singleton export (a blanket strip duplicates it →
     TS2300); merge overlapping import lines (keep-both on the same symbol → TS2300).
   - After any manual resolution: **LF-normalize** the file (`tr -d '\r'`) and `rm -rf .expo/cache/eslint`
     (mixed line-endings choke `import/namespace`, and the eslint cache serves the stale error).
3. **Mandatory re-gate after EVERY rebase, including a "clean" one** — the silent union splice is
   invisible to git and only `tsc` reveals it. When the integrated lane **fills a Phase-0 hoisted
   seam**, un-skip its contract test here (it must go skipped→green — i.e. it would have been RED
   against the no-op) and re-run every bound consumer's suite against the real impl.
4. **`git merge --ff-only`**, then **§7 verify** the merged result. Verify `git show --stat` after
   each integration commit — a `git add a b deleted-path` aborts the whole stage and silently drops
   files; amend to fold in anything dropped.
5. **Delta re-review — the merged content is not the reviewed content.** The opus·max review saw
   the pre-rebase worktree diff; everything that changed content after it escapes it. So: every
   **manual hotspot resolution** logged in step 2 (a keep-HEAD/keep-both call is semantics, and
   `tsc` proves syntax, not that the kept singleton still clears lane B's slice), and every
   **orchestrator-authored code commit** (the seam-fix-once, a shim deletion, dep glue), gets a
   **targeted top-level `bmad-code-review` pass (`model:'opus'`, effort max** — the last
   adversarial net applies to the last content change too**)** with the diff + the resolution
   rationale, before the wave closes. Record each pass in the run-record's *Delta re-review* row — an empty row must mean "no
   manual resolutions, no orchestrator code commits", never "skipped".
6. **Junction-safe teardown.** Remove the **junction LINK only** (`[System.IO.Directory]::Delete(link, $false)`
   / `cmd /c rmdir` on Windows; on POSIX the symlink itself — `rm <wt>/node_modules`, never `rm -rf`
   through it) **BEFORE** `git worktree remove --force` — or removal recurses through the
   junction and deletes the real `node_modules`. Use explicit absolute paths, not a scripted
   accumulating loop. **Verify the main `node_modules` count is UNCHANGED** after teardown (a drop
   means the real install was recursed-and-deleted).
7. **Orchestrator infra commits** on main (small, `chore(orchestration): …`):
   - the **state file** — you own `{implementation_artifacts}/sprint-status.yaml` 100%; update it in
     **bmm's vocabulary ONLY**: each integrated story key → `done`; `epic-X` → `in-progress` on its
     first merged story, `done` when all its stories are; bump `last_updated`; **preserve ALL
     comments and structure including the STATUS DEFINITIONS block** — and where that block and
     this list differ, **the file wins**. Never invent a status value (`merged`, `integrated`):
     bmm's create-story / dev-story / retrospective parse this file, and a nonstandard value makes
     dev-story re-implement an already-merged story.
   - any **lane-declared dependency** install (run the real `npx expo install` / `npm install`,
     **delete the lane's ambient `.d.ts` shim**, and **verify the installed package's real export
     surface** — read its docs/peers *before* the install; the real API can differ from the lane's
     assumption), and any post-merge cleanup. When N lanes worked around one seam N different ways,
     fix the **seam once** on main rather than reconciling N workarounds (then delta re-review it).
   - the **deferred-work ledger** — fold the wave's defer-findings (from the reviews'
     StructuredOutputs) and the critic's S4 flags into `{output_folder}/pwo/deferred-work.md`,
     append-only, one stable id per item (`DW-{wave}-{n}`). S4 seeds its consolidation work-list
     from this ledger; an item that lives only in a run-record's prose is an item S4 loses.

---

## End-of-wave critic (you; internal — proactive, not reactive)

Isolation makes each lane blind to the others, so after the wave is merged, sweep for what no single
lane could see. The critic's context has deliberately never read a lane's code (StructuredOutputs
and diffStats only), so **suspicion cannot come from vibes — delegate the detection, keep the
verdict**:

- **Cross-lane duplications — run the duplication-scout.** Spawn a fresh agent (`model:'sonnet'`,
  effort high — enumeration and clustering, not judgement; the verdict stays yours) on the
  wave's merged range (`git diff <baseline_main_head>..<final_main_head>`), instructed to:
  **(a)** enumerate every NEW exported symbol (name · path · one-line semantics); **(b)** cluster
  near-duplicates by name/signature/semantics against BOTH the wave's other new exports AND the
  pre-existing exports on main (a re-implementation of something an **earlier wave** already
  shipped is the cross-wave case — the survey-before-you-specify should have caught it; this is
  the net behind the net); **(c)** include duplicated **test fixtures/builders** and contradictory
  seed assumptions (two lanes seeding incompatible domain data); and return candidate pairs as a
  StructuredOutput (a token-level clone scan like `jscpd` is a useful *secondary* signal — it
  misses independent re-implementations, which share semantics, not tokens). Then prove each
  suspected equality with a **direct on-master cross-check** (a temp test seeding one identical
  dataset, asserting the two implementations agree), delete the temp test, and flag a **permanent
  parity test + a shared helper** for the closeout consolidation wave (S4) — as a `DW-` entry in
  the deferred-work ledger, not as prose.
- **Mocked-away policies** — a mandatory policy disabled in every test (the review's "tested or
  mocked away?" applied wave-wide).
- **Moved forward-seams** — a seam an earlier wave hoisted whose position no longer fits its now-real
  consumers.
- **Cross-lane idiom divergence** — sample this wave's new modules **side by side** (error
  handling, state management, naming, module layout). Per-lane review can be internally consistent
  and still diverge from its neighbors; only this sweep sees them together. Where lanes diverged,
  name the canonical idiom, fix trivially now or route to S4 (a `DW-` entry carrying the canonical
  idiom) — otherwise every later wave composes N dialects through adapter shims.

Fix a trivial finding now; **delegate any non-trivial change to a fresh agent** (a paste-ready,
self-contained prompt — keep the orchestrator thin); route a cross-cutting one to S4's de-parallelized
consolidation wave via the ledger.

---

## End-of-wave smoke

The gate follows the wave's **content, not its tag** — a mixed wave (engines + screens) runs
**both** gates:

- **Any engine/computation lane in the wave** → **jest + the empirical sweep of the MERGED result**
  (the playbook names the range and the runner). The sweep is **executed, never narrated** —
  delegate it to a fresh agent (`model:'sonnet'`, effort high — execution + observation, per the
  routing table) with a pinned StructuredOutput:
  `{ range, pointsExecuted, expectationSource, mismatches:[{input,expected,observed}], outputTail, evidencePath }`,
  run on **merged main after integration** (per-lane review sweeps ran in isolated worktrees and
  cannot see the composition). PASS iff the named range is covered (endpoints + the playbook's
  density), expectations were **independently re-derived** (`expectationSource` says from what),
  and `mismatches` is empty. Paste the real `outputTail` into the run-record — **a sweep with no
  captured execution output did not happen** (the RED-proof rule, A3). A green unit suite alone is
  not the gate.
- **Anything renders** → **delegate to `pwo-ui-smoke` (S5)** at `model:'sonnet'`, effort high (S5
  itself says reliability comes from its capture-and-compare protocol, not reasoning depth). Pass
  it exactly the wave's **smoke criterion** from the playbook — the **journey** (ordered steps),
  the **reference mockup(s)**, and the **expected live figures** — plus the runtime state (the
  emulator/browser is already up; S5 is a *consumer* of running infra, never an owner). **Trust S5's StructuredOutput verdict**
  (`PASS | FAIL | BLOCKED`, with per-screen `deltas` and per-figure `match`) **without re-driving
  the app or re-reading its screenshots** — S5 owns that contract; do not re-define it. **But
  validate its envelope before accepting a PASS**: a screen-wave PASS counts **only** with
  `harness ≠ none`, `uiBlindSpotCovered:true`, non-empty `evidencePaths`, and every asserted screen
  present in `screensVisited` — anything else is treated as **BLOCKED** and routes to triage (a
  jest-only degrade must never silently clear a screen gate; that is the exact blind spot S5
  exists to cover). A `FAIL` (a blocking/major delta or a figure mismatch) or a `BLOCKED` routes
  to **Smoke triage** below, not into the handoff prompt; a smoke that asserts a screen whose nav
  entry lands in a later wave returns it under `renderDeferred` — expected, not a failure.

Record the verdict (and S5's `evidencePaths` / the sweep's `outputTail`) in the run-record.

---

## Smoke triage (mid-build — the FAIL/BLOCKED path)

The smoke runs **after** integration: the wave's code is already on main, worktrees torn down —
so triage is repair-on-main, never a lane-rollback reflex, and never an improvised unreviewed
hotfix. Classify each delta / figure-mismatch / crash:

- **Trivial + targeted** (a single-screen delta, a one-line figure fix, self-contained) →
  delegate a hotfix to a fresh agent (`model:'opus'`, effort high), **code-review the patch
  top-level (opus·max)**, re-gate, **re-smoke via S5** — bounded at ~2 hotfix+re-smoke cycles,
  then escalate to the human.
- **Non-trivial / cross-cutting** → escalate to the human with the explicit options:
  **revert the offending lane** (a `chore(orchestration)` revert commit — forward-only, never
  rewrite history) · **repair as a lane in the next wave** (add it to the handoff's lane list,
  carded like any lane) · **accept as recorded debt** (a `DW-` entry in the deferred-work ledger
  + the run-record).
- **BLOCKED** (the harness could not come up, or S5's envelope failed validation) → fix the
  runtime-ownership problem and re-smoke. A BLOCKED never converts to PASS by re-reading the
  screenshots or by narrowing the journey.

The **handoff prompt is emitted ONLY after a re-smoke PASS or an explicit human acceptance** — and
it must carry the accepted debt. Record everything in the run-record's *Smoke triage* table; a
wave that closes on accepted debt rather than a clean PASS is `overall: triaged`, not
`integrated`.
