# Parallel Wave Orchestration (PWO)

**Version 1.0.0**

> A BMad module that implements a backlog **in parallel** — many concurrent lanes,
> one git worktree each — while a single orchestrator keeps the main branch **always
> green and conflict-free**.

PWO is an **expansion of `bmm`**. It slots into the BMad lifecycle **after planning**
(prd → architecture → epics → readiness) and **before the first create-story**, and runs
through to closeout. `create-story` / `dev-story` are **emulated inside** the wave
workflows (parallel); `code-review` and integration run **outside** (top-level / serial).
The orchestrator is the **only writer** of the main branch and the **only owner** of the
coordination file (`sprint-status`).

It was distilled from a full-scale battle test (a complete app build: Phase 0 + 5 waves,
31 lanes, 10 epics) that delivered **0 main-branch corruptions over 31 lanes**, no test
loss across 31 fast-forward merges, no schema corruption, an on-device smoke PASS, and a
real **~4–5×** speedup.

## The six workflows (S0–S5)

| Skill | Role |
| ----- | ---- |
| **`pwo-probe-harness`** (S0) | Empirically probe what the harness allows — the harness Facts (F1-F5) — so planning rests on fact, not assumption. **Required first step.** |
| **`pwo-plan-waves`** (S1) | Turn epics + architecture into a safe playbook: the **applicability gate** (GO/NO-GO), an adversarially-verified dependency DAG, the wave plan, the Phase 0 spec, and per-lane cards. |
| **`pwo-build-phase0`** (S2) | Build **and prove** the Phase-0 guard-rails on main before wave 1 (migration-set guard proven RED, hoisted seams, `.gitattributes`, pre-provisioned deps, test-ids split per screen). |
| **`pwo-run-wave`** (S3) | Execute **one** wave end-to-end keeping main green: emulate create/dev → gate → verify-real-git → top-level review → serial integration → critic → smoke → Field Note + handoff. Invoked once per wave, in a fresh session. |
| **`pwo-closeout`** (S4) | Close out the backlog: a final E2E smoke campaign by user journey, a **de-parallelized** consolidation wave, and a maturation report that feeds the suite's own improvement. |
| **`pwo-ui-smoke`** (S5) | On-call leaf: smoke a rendered screen/journey **against its mockup** and return a compact verdict. Delegated by S3 (end-of-screen-wave) and S4 (final E2E). |
| `pwo-setup` | Module setup: records config, verifies prerequisites, registers the capabilities. |

Lifecycle:

```
readiness ─► S0 probe ─► S1 plan ─► S2 phase-0 ─► S3 run-wave ×N ─► S4 closeout ─► bmad-retrospective
                                                       │
                                                       └─ calls S5 (ui-smoke)  ◄── also called by S4
```

## When to use it — and when not to

**Use it** for a large backlog whose dependency DAG is genuinely **wide** (≥ ~4–5
parallelizable lanes), on a codebase that satisfies the two entry pre-conditions:

- **P1 — a disciplined architecture**: layers / ports / single-source / centralized
  registries (so lanes don't collide).
- **P2 — whole-app mockups _and_ a verified UI smoke harness** (a green CI renders no
  pixels; the smoke is the only counter-power to the UI blind spot).

**Don't use it** for short, linear backlogs, or projects lacking P1/P2 — the
orchestration overhead never pays back. The applicability gate in `pwo-plan-waves`
enforces this and a NO-GO ("build sequentially") is a valid, useful outcome.

## Requirements

- **git with worktree support** (the atom of lane isolation; a non-shallow clone is
  recommended).
- A **dynamic-orchestration / Workflow tool** whose sub-agents have at least
  `{shell, Read, Write, Edit, Skill, git}` (`Skill` is used only by the S0 leak-probe; lanes
  deliberately emulate and never invoke it).
- A top-level **`Agent`/code-review** harness (code-review fans out and runs top-level).
- A **UI smoke harness**, project-dependent: `expo-mcp` | `chrome-devtools` | `adb`
  (or `none`, which leaves the UI blind spot uncovered).
- The **`bmm`** module — PWO reuses (never reinvents) `create-story` / `dev-story`
  (emulated), `code-review`, `retrospective`, and `sprint-status`.
- **Python 3.9+** (with `uv` recommended, else `pyyaml` installed) — `pwo-setup`'s config-merge
  scripts need it.

## Install

**First, install `bmm`** — PWO expands it. If your project doesn't already have it:

```bash
npx bmad-method install --tools claude-code   # select bmm
```

Then install PWO from any git URL, via the BMad installer (discovery mode reads
`.claude-plugin/marketplace.json`):

```bash
npx bmad-method install --custom-source https://github.com/zRawday/bmad-pwo --tools claude-code --yes
```

The same `.claude-plugin/marketplace.json` also works with Claude Code's native plugin
marketplace (experimental — untested until the repo is published; the `bmad-method install
--custom-source` route above is the supported one):

```
/plugin marketplace add https://github.com/zRawday/bmad-pwo
/plugin install pwo@bmad-pwo
```

Then run setup to record configuration and register the capabilities:

```
/pwo-setup
```

## Configuration (collected by `pwo-setup`)

| Variable | Meaning | Default |
| -------- | ------- | ------- |
| `worktree_workspace` | Sibling directory that will hold the lane worktrees | `{project-root}/../<project-name>-wt` |
| `main_branch` | The branch the orchestrator keeps green | `master` (detected from git) |
| `smoke_harness` | UI smoke target (`expo-mcp` \| `chrome-devtools` \| `adb` \| `none`) | `expo-mcp` |

`pwo-setup` is **additive and non-destructive**: it writes the `pwo` config section and
appends the help rows; it never deletes another module's config. It verifies git worktree
support (and warns on a shallow repo), probes the configured smoke harness (P2), and
records that **`pwo-probe-harness` (S0) is the required first step** — but does **not**
auto-run it (the probe is human-triggered, right after readiness).

## Quick start

1. `/pwo-setup` — configure + verify prerequisites.
2. `/pwo-probe-harness` — characterize the harness (run it by hand; S1 refuses to plan
   without a clean `harness-profile`).
3. `/pwo-plan-waves` — applicability gate + DAG + waves + Phase 0 spec.
4. `/pwo-build-phase0` — ship & prove the guard-rails on main.
5. `/pwo-run-wave` — once per wave, in a fresh session; paste the previous wave's handoff
   prompt to continue.
6. `/pwo-closeout` — final smokes + consolidation + maturation report, then
   `bmad-retrospective`.

> **Guided continuity.** Each step ends by printing a **paste-ready prompt** (command-first) for the
> next step — copy it into a **fresh** Claude session to continue, which keeps each step's context
> budget clean. You are never left guessing what to run. `pwo-ui-smoke` is **auto-invoked** by steps
> 5–6; you never launch it yourself. The chain:
> `pwo-setup → pwo-probe-harness → pwo-plan-waves → pwo-build-phase0 → pwo-run-wave ×N → pwo-closeout → bmad-retrospective`.

## Design notes

- **Verify from the source of truth**, never a self-report (real git, real code, the state
  file).
- **The orchestrator coordinates, it does not implement** — it delegates everything bulky
  and keeps only the targeted, load-bearing judgement (the gate).
- **A single writer to the state file** — the orchestrator owns `sprint-status` 100%;
  lanes never touch it.
- **Emulate create/dev** — never invoke the real Skill (it would leak to the main repo).
- **Main green > speed > cost** — effort routes *upward*; token economy is not a goal.

## License

[MIT](./LICENSE) © 2026 zRawday
