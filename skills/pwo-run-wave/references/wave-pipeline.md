# Wave Pipeline — the orchestrator's recipe for one wave

The per-step recipe SKILL.md routes here for: the **effort table**, the **WF1/WF2 script
skeletons + StructuredOutput schemas** (with the emulate / leak-guard / footprint-only parade),
the **gate fact-extraction recipe**, the **§7 real-git verification checklist**, the **serial
integration + junction-safe teardown** order, the **end-of-wave critic** checklist, and the **S5
smoke call** wiring. Read the section for the step you are running. This file stands alone — it
does not assume SKILL.md is still in context.

`REPO` = the absolute main-repo path · `WT` = the absolute `worktree_workspace` · `MAIN` =
`main_branch`. `LANES` comes from this wave's per-lane cards in the playbook (`pwo-plan-waves`):
each card carries `key`, `title`, `footprint`, `hotspots`, the **⚙ constraint verbatim**, and a
`tier` (`trivial | standard | critical`).

---

## The effort table (read while constructing every `agent()` call)

Route effort **upward** — the cap goes on whatever *judges* or *codes at stakes*; drop back only on
the purely procedural. **Token economy is not a goal** (main green > speed > cost).

| Task | Effort | Why |
| ---- | :----: | --- |
| create (emulate) | **high** | the gate catches content, but a poor spec burns the gate's scarce budget |
| dev (emulate) | **xhigh** | single pass, no retry net → max capability first try; **`max` on a `critical` lane** (tiering upward is the only safe direction) |
| code-review | **max** | the last adversarial net; it fights the dev's confirmation bias |
| patch (load-bearing) | **xhigh** | subtle coding (rounding, FK, boundary) |
| smoke / mechanical | **high** | bounded by observation fidelity + ops robustness, not reasoning depth — don't overthink the procedural |

The lane's `tier` (from S1's card) does two concrete jobs at runtime: it **arms the anti-mis-tiering
guard at the gate** (a `critical` lane whose spec reads thin gets sent back) and it **bumps
`critical` lanes to `dev=max`**.

---

## WF1 — create (parallel, EMULATE)

One workflow agent per lane. The load-bearing steps are **emulate-not-invoke**, **absolute
worktree paths**, an **explicit spec-only commit**, and a **main-repo leak-guard**. Cut every
lane's worktree from the **same** main snapshot (commit nothing to main between lane creations —
asymmetric bases are harmless but the integration rebase has to fix them).

```js
export const meta = { name: 'waveN-create', description: 'emulate create-story per lane', phases: [{ title: 'Create' }] }
const REPO = '<abs main repo>', WT = '<abs worktree_workspace>', MAIN = '<main_branch>'
const LANES = [ /* { key, slug, titleHint, constraint:'<the card ⚙ constraint, verbatim>' }, … */ ]
const SCHEMA = { type:'object', additionalProperties:false, properties:{
  storyKey:{type:'string'}, branch:{type:'string'}, specPath:{type:'string'}, specCommitSha:{type:'string'},
  specOnlyCommit:{type:'boolean'}, mainRepoClean:{type:'boolean'}, acHeadlines:{type:'string'},
  filesTouched:{type:'string'}, notes:{type:'string'} },
  required:['storyKey','branch','specPath','specCommitSha','specOnlyCommit','mainRepoClean','acHeadlines','filesTouched','notes'] }
function buildPrompt(l){ const wt = `${WT}/${l.key}`; return `Autonomous create lane for Story ${l.key} (${l.titleHint}).
0) BASE = git -C "${REPO}" rev-parse HEAD (the main branch; leave it UNTOUCHED).
1) git -C "${REPO}" worktree add "${wt}" -b story/${l.key} ${MAIN}  (retry 2-3x on a .git lock; reuse if it exists).
2) EMULATE create-story — do NOT invoke the Skill tool (it writes to the MAIN repo and leaks). Read the epic's story +
   an in-repo example spec as the FORMAT, and AUTHOR THE SPEC DIRECTLY to ${wt}/<impl-artifacts-dir>/${l.slug}.md,
   baking this constraint VERBATIM: "${l.constraint}". Reads from ${REPO} are fine; WRITE only under ${wt}.
3) Commit ONLY the spec (never git add -A): git -C "${wt}" add "<impl-artifacts-dir>/${l.slug}.md"
   && git -C "${wt}" commit -m "docs(story ${l.key}): create story spec".
4) LEAK-GUARD: assert git -C "${REPO}" rev-parse HEAD == BASE, and git -C "${REPO}" diff --quiet && diff --cached --quiet
   (ignore known pre-existing untracked files; clean only your own leak). Report mainRepoClean + specOnlyCommit.
Return the schema; filesTouched MUST be exactly the one spec file; acHeadlines = the spec's AC one-liners.` }
phase('Create')
return { specs: (await parallel(LANES.map(l => () =>
  agent(buildPrompt(l), { label:`create:${l.key}`, phase:'Create', schema:SCHEMA, effort:'high' })))).filter(Boolean) }
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
2. **Verify the spec is dev-ready:** the ⚙ constraint is baked verbatim; the footprint matches the
   card; the ACs match the epic.
3. **Cross-check every load-bearing value** (seeds, barèmes, fiscal constants) against the source of
   truth yourself — never let a `critical` lane carry an unverified number.
4. **Anti-mis-tiering guard:** a lane the card tagged `critical` whose spec reads thin is under-spec'd
   — send it back (and it runs `dev=max`).
5. **Amend in-worktree, surgically.** Where the gate resolves something — a thin spec, a moved seam,
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
const SCHEMA = { type:'object', additionalProperties:false, properties:{
  storyKey:{type:'string'}, branch:{type:'string'}, gatesPassed:{type:'boolean'}, testCount:{type:'number'},
  diffStat:{type:'string'}, featCommitSha:{type:'string'}, footprintOnly:{type:'boolean'}, mainRepoClean:{type:'boolean'},
  amendmentHonored:{type:'boolean'}, autocommitHandling:{type:'string'}, notes:{type:'string'} },
  required:['storyKey','branch','gatesPassed','testCount','diffStat','featCommitSha','footprintOnly','mainRepoClean','amendmentHonored','autocommitHandling','notes'] }
function buildPrompt(l){ const wt = `${WT}/${l.key}`; return `Autonomous dev lane for Story ${l.key}. Worktree ${wt} on story/${l.key} holds the gated spec.
0) BASE = git -C "${REPO}" rev-parse HEAD (leave main UNTOUCHED). cd "${wt}" so cwd-relative commands stay local.
1) Junction node_modules: link ${wt}/node_modules → ${REPO}/node_modules. NEVER install through the junction.
2) EMULATE dev-story: implement DIRECTLY from the spec's Tasks/ACs (honor the top-of-spec ⚙ GATE AMENDMENT if present,
   and any named in-repo pattern) to ABSOLUTE ${wt}/ paths. Honor every footprint / hotspot / purity / deferred-import
   rule. Do NOT invoke the Skill tool.
3) Gates IN THE WORKTREE: typecheck && lint && test (all against ${wt}); fix under footprint until green; capture testCount.
4) COMMIT explicitly, footprint-only (NEVER git add -A): git -C "${wt}" add <footprint files>
   && commit -m "feat(story ${l.key}): <subject>". If a stray autocommit ever fires, normalize to one footprint-only
   feat commit (amend out any coordination-file/stray paths).
5) LEAK-GUARD (as WF1): main HEAD == BASE, tracked tree clean. Report footprintOnly + mainRepoClean + amendmentHonored + autocommitHandling.
Return the schema (diffStat = git diff --stat <specCommit> HEAD).` }
phase('Dev')
return { devs: (await parallel(LANES.map(l => () =>
  agent(buildPrompt(l), { label:`dev:${l.key}`, phase:'Dev', schema:SCHEMA, effort: l.tier === 'critical' ? 'max' : 'xhigh' })))).filter(Boolean) }
```

---

## §7 — verify each lane with REAL git (before any merge)

Workflow agents **self-report**; re-check from the source of truth. Across a full build the
self-reports matched real git 100% — and this re-check stays non-negotiable, because it is how you
catch a parallelization defect the gates pass green. Per lane:

- **Footprint compliance** — `git -C <wt> diff --stat <specCommit> HEAD` touches **only** the card's
  files. A stray file = scope creep or a mis-routed edit; investigate before merge.
- **Hotspot discipline** — exactly one new migration object at its **pre-allocated** version;
  append-only labelled blocks on registries/barrels; a non-owner of a co-composed screen edits only
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
them yourself as **Agent calls** in the main loop. Effort **max**. Give each reviewer the **diff
range**, the **spec**, the three lenses (correctness / edge-cases / acceptance), the lane's
invariants, and **authority to apply safe patches in the worktree + re-gate** (a load-bearing patch
is xhigh). The prompts that earn the review its keep:

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
   invisible to git and only `tsc` reveals it.
4. **`git merge --ff-only`**, then **§7 verify** the merged result. Verify `git show --stat` after
   each integration commit — a `git add a b deleted-path` aborts the whole stage and silently drops
   files; amend to fold in anything dropped.
5. **Junction-safe teardown.** Remove the **junction LINK only** (`[System.IO.Directory]::Delete(link, $false)`
   / `cmd /c rmdir`) **BEFORE** `git worktree remove --force` — or removal recurses through the
   junction and deletes the real `node_modules`. Use explicit absolute paths, not a scripted
   accumulating loop. **Verify the main `node_modules` count is UNCHANGED** after teardown (a drop
   means the real install was recursed-and-deleted).
6. **Orchestrator infra commits** on main (small, `chore(orchestration): …`): the **state file** (you
   own `sprint-status` 100% — update it to reflect *merged* reality), any **lane-declared dependency**
   install (run the real `npx expo install` / `npm install`, **delete the lane's ambient `.d.ts`
   shim**, and **verify the installed package's real export surface** — read its docs/peers *before*
   the install; the real API can differ from the lane's assumption), and any post-merge cleanup. When
   N lanes worked around one seam N different ways, fix the **seam once** on main rather than
   reconciling N workarounds.

---

## End-of-wave critic (you; internal — proactive, not reactive)

Isolation makes each lane blind to the others, so after the wave is merged, sweep for what no single
lane could see:

- **Cross-lane duplications** — N lanes that independently re-implemented one formula/predicate
  because no shared home exists yet. Prove suspected equality with a **direct on-master cross-check**
  (a temp test seeding one identical dataset, asserting the two assemblers agree), then delete it and
  flag a **permanent parity test + a shared helper** for the closeout consolidation wave (S4).
- **Mocked-away policies** — a mandatory policy disabled in every test (the review's "tested or
  mocked away?" applied wave-wide).
- **Moved forward-seams** — a seam an earlier wave hoisted whose position no longer fits its now-real
  consumers.

Fix a trivial finding now; **delegate any non-trivial change to a fresh agent** (a paste-ready,
self-contained prompt — keep the orchestrator thin); route a cross-cutting one to S4's de-parallelized
consolidation wave.

---

## End-of-wave smoke

- **Foundation-wave** (pure engines, nothing on screen) → the gate is **jest + the empirical sweep**
  of the realistic input range (the playbook names the range). A green unit suite alone is not the
  gate.
- **Screen-wave** (something renders) → **delegate to `pwo-ui-smoke` (S5)**. Pass it exactly the
  wave's **smoke criterion** from the playbook — the **journey** (ordered steps), the **reference
  mockup(s)**, and the **expected live figures** — plus the runtime state (the emulator/browser is
  already up; S5 is a *consumer* of running infra, never an owner). **Trust S5's StructuredOutput
  verdict** (`PASS | FAIL | BLOCKED`, with per-screen `deltas` and per-figure `match`) **without
  re-driving the app or re-reading its screenshots** — S5 owns that contract; do not re-define it. A
  `FAIL` (a blocking/major delta or a figure mismatch) or a `BLOCKED` (the harness could not come up)
  routes to triage, not into the handoff prompt; a smoke that asserts a screen whose nav entry lands
  in a later wave returns it under `renderDeferred` — expected, not a failure.

Record the verdict (and S5's `evidencePaths`) in the run-record.
