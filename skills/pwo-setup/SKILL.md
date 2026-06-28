---
name: "pwo-setup"
description: Sets up the Parallel Wave Orchestration (PWO) module in a project. Use when the user requests to 'install pwo module', 'configure Parallel Wave Orchestration', or 'setup PWO'.
---

# PWO Module Setup

## Overview

Installs and configures the **Parallel Wave Orchestration (PWO)** module — an **expansion of `bmm`** —
into a project. PWO is six workflow skills (S0–S5) plus this setup; it reuses `bmm`'s
`create-story` / `dev-story` (emulated), `code-review`, `retrospective`, and `sprint-status`, so
**this setup installs no other skills** — it only records configuration and registers the PWO
capabilities for the help system.

Module identity (name, code, version) comes from `./assets/module.yaml`. Setup is **additive and
non-destructive**: it writes the `pwo` section of config and appends the PWO help rows, and never
deletes another module's config or the installer's `_bmad/_config/` manifests. It writes:

- **`{project-root}/_bmad/config.yaml`** — shared project config: core settings at root (e.g.
  `output_folder`, `document_output_language`) plus a `pwo:` section with the module's three
  variables (`worktree_workspace`, `main_branch`, `smoke_harness`). User-only keys (`user_name`,
  `communication_language`) are **never** written here.
- **`{project-root}/_bmad/config.user.yaml`** — personal settings intended to be gitignored:
  `user_name`, `communication_language`, and any module variable marked `user_setting: true` in
  `./assets/module.yaml` (PWO has none by default).
- **`{project-root}/_bmad/module-help.csv`** — registers the PWO capabilities for the help system.

The config and help scripts use an anti-zombie pattern — existing `pwo` entries are removed before
writing fresh ones, so a re-run never leaves stale values.

`{project-root}` is a **literal token** in config _values_ (the data written into the files above) —
never substitute it there; it signals to the consuming LLM that the value is relative to the project
root. **This does not apply to the filesystem path _arguments_ passed to the scripts below** (the
`--*-path`, `--*-dir`, and `--target` arguments): those are real paths, so you **must** resolve
`{project-root}` to the actual project root before running, or the scripts will write to a literal
`{project-root}/` directory under the skill folder. The scripts reject an unresolved token with an
error.

> **Tooling note.** The commands below call `python3`. If `python3` is not on PATH (common on
> Windows), use `python` instead — both invoke the same interpreter.

## On Activation

1. Read `./assets/module.yaml` for module metadata and variable definitions (the `code` field, `pwo`,
   is the module identifier).
2. Check if `{project-root}/_bmad/config.yaml` exists and already has a `pwo:` section — if so, inform
   the user this is a **reconfigure** (update); otherwise it is a **fresh install**.
3. **Non-destructive legacy read.** If `{project-root}/_bmad/core/config.yaml` exists (this repo may
   still use the legacy per-module layout), **read it for default values only** — in particular
   preserve any existing `communication_language`, `user_name`, and `output_folder`. Do **not** delete
   it and do **not** consolidate other modules; PWO setup only adds the `pwo` section. (BMad's full
   legacy→consolidated migration is a separate concern handled by the core/bmm setup.)

If the user provides arguments (e.g. `accept all defaults`, `--headless`/`-H`, or inline values like
`smoke harness is adb, main branch is main`), map provided values to config keys, use defaults for the
rest, and skip interactive prompting. Still run the prerequisite checks and display the full
confirmation summary at the end.

## Verify Prerequisites (PWO)

PWO's atom of isolation is the **git worktree**, so confirm it before configuring.

1. **git worktree support.** Run `git -C {project-root} worktree list`. If it succeeds, support is
   present (note the existing worktrees). If it errors, git is too old or this is not a git repo —
   tell the user PWO cannot run without worktree support and stop after recording config.
2. **Shallow-repo warning.** Run `git -C {project-root} rev-parse --is-shallow-repository`. If it
   prints `true`, **warn** the user: a shallow clone can make the rebase/merge steps in `pwo-run-wave`
   unreliable; recommend `git fetch --unshallow` before running waves. (Worktrees still function — this
   is a warning, not a blocker.)
3. **Detect the main branch.** Run `git -C {project-root} rev-parse --abbrev-ref HEAD` (fall back to
   `git symbolic-ref --short refs/remotes/origin/HEAD` if detached). Use the result as the **default
   for `main_branch`** in the next step instead of the literal `master` from `module.yaml`.

## Collect Configuration

Ask the user for values. Show defaults in brackets. Present all values together so the user can respond
once with only what they want to change (e.g. "smoke harness none, rest are fine"). Never tell the user
to "press enter" or "leave blank" — in a chat interface they must type something to respond.

**Default priority** (highest wins): existing `pwo` config values > legacy `_bmad/core` values (read
non-destructively above) > `./assets/module.yaml` defaults.

**Core config** (only if no core keys exist yet at the root of `config.yaml`): `user_name`
(default: BMad), `communication_language` and `document_output_language` (default: English — ask as a
single language question; both keys get the same answer), `output_folder`
(default: `{project-root}/_bmad-output`). If legacy `_bmad/core/config.yaml` already defines these, use
those as the defaults so you don't override the user's existing choices. `user_name` and
`communication_language` are written exclusively to `config.user.yaml`; the rest go to `config.yaml` at
root.

**Module config** (the three `pwo` variables from `./assets/module.yaml`):

- **`worktree_workspace`** — the sibling directory that will hold the lane worktrees. The default is
  `{project-root}/../<project-name>-wt`; **resolve `<project-name>` to the actual project folder name**
  (the basename of `{project-root}`) before presenting it, e.g. `{project-root}/../Gesto-wt`. Accept an
  absolute path or a `{project-root}`-relative one; keep the `{project-root}` token literal in the
  stored value.
- **`main_branch`** — default to the branch detected in *Verify Prerequisites* (else `master`).
- **`smoke_harness`** — a single choice from: `expo-mcp` (React Native / Expo), `chrome-devtools`
  (web), `adb` (raw Android), or `none` (no UI harness). Default `expo-mcp`. If the user picks `none`,
  remind them screen-wave validation will be **jest-only** and the **UI blind spot stays uncovered**.

## Write Files

Write a temp JSON file with the collected answers structured as `{"core": {...}, "module": {...}}`
(omit `core` if core keys already exist at the root of `config.yaml`). Values inside this JSON keep the
literal `{project-root}` token. Then run both scripts — they can run in parallel since they write to
different files.

These commands **omit `--legacy-dir` on purpose**: PWO setup must not delete `_bmad/core/config.yaml`
or any other module's legacy config. Legacy values were already folded in as defaults during
collection. Replace `{project-root}` in every path argument with the actual project root before running
— these are filesystem paths, not config values. Leave `{temp-file}` and `pwo` as-is.

```bash
python3 ./scripts/merge-config.py --config-path "{project-root}/_bmad/config.yaml" --user-config-path "{project-root}/_bmad/config.user.yaml" --module-yaml ./assets/module.yaml --answers {temp-file}
python3 ./scripts/merge-help-csv.py --target "{project-root}/_bmad/module-help.csv" --source ./assets/module-help.csv
```

Both scripts output JSON to stdout. If either exits non-zero, surface the error and stop.
`merge-config.py` reports `module_keys` and `user_keys`; `merge-help-csv.py` reports `rows_added` and
`total_rows`. Run either with `--help` for full usage.

## Verify the Smoke Harness (P2)

PWO's pre-condition **P2** requires a *verified* UI smoke harness (a green CI renders no pixels). Do a
quick **probe-style** check of the configured `smoke_harness` and **record** the result (do not hard-fail
— record what you observe):

- **`expo-mcp`** — confirm the Expo MCP tooling is reachable (the `mcp__expo__*` tools, or `npx expo`
  in the project). Record reachable / not-reachable.
- **`chrome-devtools`** — confirm the Chrome DevTools MCP (`mcp__chrome-devtools__*`) is available.
- **`adb`** — run `adb devices`; record whether `adb` is on PATH and whether a device/emulator is
  attached.
- **`none`** — record `none` and **warn**: screen-wave validation will be jest-only and the UI blind
  spot is uncovered. `pwo-ui-smoke` (S5) will degrade to a no-UI verdict.

This mirrors the harness probe's spirit (observe, don't assume) but stays lightweight; the full
empirical characterization is `pwo-probe-harness` (S0), the required first step (below).

## Create Output Directories

Resolve `{project-root}` to the actual project root for filesystem operations only (the stored config
values keep the literal token). Create, if missing:

- `output_folder` (e.g. `{project-root}/_bmad-output`), and
- `{output_folder}/pwo/` — where the PWO artifacts land (harness profile, playbook, receipts, records,
  maturation report).

**Do not** auto-create `worktree_workspace` here — handle it in the next (optional) step.

## Optionally Scaffold the Worktree Workspace

Ask the user whether to create the `worktree_workspace` directory now. If yes, resolve its value
(substituting the real project root for `{project-root}`) and `mkdir -p` it. If no, leave it — the
worktrees will be created on demand by `pwo-build-phase0` / `pwo-run-wave`. Never create anything inside
the main repo tree for this; the workspace is a **sibling** of the project.

## No Legacy Cleanup

There is **no installer package directory to clean** for PWO: its skills live in `{project-root}/skills/`
and `{project-root}/.claude/skills/`, not in a `_bmad/pwo/` package. Do **not** run a cleanup that
removes `_bmad/_config/` or other modules' directories — those hold active BMad manifests and config.
(This is why the bundled `cleanup-legacy.py` is intentionally not invoked by this setup.)

## Record the Required First Step

PWO does **not** auto-run the harness probe. Record clearly for the user that **`pwo-probe-harness`
(S0) is the REQUIRED first step** and is **human-triggered** — to be run *now*, right after readiness
and before `pwo-plan-waves` (S1). S1 refuses to freeze the parallelization mechanism without a clean
`harness-profile`, so the probe gates the whole pipeline. Do not invoke it from here.

## Confirm

Use the script JSON output to display what was written: the `pwo` config values (in the `pwo:` section
of `config.yaml`), any user settings (`config.user.yaml`), and the help rows added (`rows_added`).
State whether this was a fresh install or a reconfigure. Summarize the prerequisite results (worktree
support; shallow warning if any; detected main branch), the smoke-harness probe result, whether the
worktree workspace was created, and the **required first step** (`pwo-probe-harness`). Then display the
`module_greeting` from `./assets/module.yaml`.

## Outcome

Once the user's `user_name` and `communication_language` are known (from collected input, arguments, or
existing config), use them consistently for the rest of the session: address the user by their
configured name and communicate in their configured `communication_language`.
