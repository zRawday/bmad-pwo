# Probe Catalog — the factored, reloadable probe logic

This file is the reusable core of `pwo-probe-harness`: one entry per harness Fact, each
carrying the **observe-recipe** a probe subagent executes, the **safety** rule that keeps
the host repo intact, and the **verdict rubric** that maps what was observed to
`clean | partial | broken`. SKILL.md reads this file to author each probe subagent's
prompt and StructuredOutput schema. The **lever-6 failure-probe** reuses the same
recipe/verdict shape — keep it generic and self-contained.

## The non-negotiable every probe carries

Bake these into every probe subagent prompt, verbatim in spirit:

- **Report what you observe by executing. Never guess, never reason from plausibility.**
  A determination not backed by a captured command output, file content, tool list, or
  error string does not exist. If you could not observe it, say so — that is `broken`,
  not a guess.
- **Ground-truth or nothing.** `observations` holds the raw evidence (exact tool lists,
  absolute paths, literal stdout/stderr, the actual merged bytes). `determination` is the
  factual answer derived *only* from those observations.
- **Leave the host repo exactly as you found it.** Probes that touch a real repo capture a
  baseline first and restore it last; probes that exercise generic git behavior do it in a
  throwaway temp repo, never the project repo.

## The shared verdict schema (every probe returns this)

Every probe subagent is forced to a StructuredOutput object with these fields:

```
factId         string   "F1".."F5"
probeRan       boolean   did the probe execute end-to-end?
applicable     boolean   false only when a required input was absent (e.g. F5 with no dep) -> recorded, NOT escalated
observations   string    RAW ground truth — tool lists, paths, literal stdout/stderr, merged bytes. No interpretation.
determination  string    the factual answer to the Fact's question, derived only from observations
verdict        string    "clean" | "partial" | "broken"
verdictReason  string    one line, against this Fact's rubric below
safetyRestored boolean   true if the host repo was restored to its captured baseline (probes that touch a real repo); true by default for temp-repo probes
notes          string    anything the human router needs (the corruption shape, the mismatched export, the literal lock error)
```

## The universal confidence rule (applied by SKILL.md, not the probe)

Each Fact is probed by **two identical copies**. After both return, SKILL.md folds them:

- **clean** — both copies ran, agree on the determination, and that determination matches
  this Fact's "clean" rubric below.
- **partial** — the copies **disagree** (a non-deterministic harness is, by itself, a
  reason not to trust the Fact), or a single copy's result is conditional/incomplete.
- **broken** — a copy could not execute the probe (error, missing prerequisite, blocked).

Disagreement between copies can never be `clean`, whatever either copy claimed. Any Fact
that lands `≠ clean` (and is `applicable`) is escalated to the human.

---

## F1 — Workflow-subagent toolset & nesting

**Determines:** whether a workflow subagent can spawn sub-agents — which decides whether a
fan-out skill (most importantly `code-review`) can be nested inside a workflow, or must run
top-level. (Method §2 Fact 1.)

**Hard dependency:** this Fact is *about a workflow subagent*, so the probe MUST run as one
(via the Workflow tool). It cannot be substituted by a top-level Agent — that would
characterize the wrong thing.

**Observe-recipe (the subagent executes, then reports):**
1. List your *actual* available tools, verbatim (the real toolset, not what you assume).
2. Run `ToolSearch "select:Agent"` and report whether the `Agent` tool schema loads or is refused.
3. If and only if it loads, attempt to spawn a trivial sub-agent instructed to reply `PONG`,
   and report the literal result or the literal error.

**Safety:** read-only; spawns nothing that writes. No restore needed.

**Verdict rubric:**
- `clean` — the tool list + ToolSearch result + (if attempted) spawn outcome were all
  captured, and both copies agree on the same determination (whether nesting is possible).
- `partial` — copies disagree, or ToolSearch behaves inconsistently across the two copies.
- `broken` — a copy could not enumerate its tools or the probe errored before observing.

**Determination to record (example shape):** `"no Agent tool present; ToolSearch refused; nesting impossible"` — record whatever is actually found.

---

## F2 — Emulate-vs-invoke (main-repo leak)

**Determines:** whether invoking the **real** Skill tool from inside a workflow subagent
writes into the MAIN repo (a leak) and/or fires a `git add -A` autocommit there — which
decides whether lanes must **emulate** create/dev or may invoke them. (Method §2 Fact 2.)

**Hard dependency:** must run as a workflow subagent. Requires one real single-agent skill
to invoke — default `create-story` for a throwaway story key; or a benign configured skill.
If no such skill is available, mark `applicable: false`.

**Observe-recipe:**
1. Capture the MAIN repo baseline: `git -C <repo> rev-parse HEAD` and `git -C <repo> status --porcelain`.
2. Create a throwaway worktree off the main branch.
3. From the worktree cwd, **invoke one real single-agent skill** (e.g. create-story for a
   throwaway story). Do not redirect its paths — the point is to see where it writes by default.
4. Record the **absolute path** where its output actually landed (inside the worktree, or in
   the MAIN repo), and whether an autocommit fired in the main repo (`git log` / `status`).
5. **RESTORE**: revert any main-repo change to the captured HEAD + clean state; remove the
   throwaway worktree and branch. Re-verify HEAD == baseline and the tracked tree is clean.
6. Report the landing path, the autocommit observation, and `safetyRestored`.

**Safety:** capture-then-restore is mandatory. If you cannot restore the main repo, that is
`broken` and must be flagged loudly in `notes`.

**Verdict rubric:**
- `clean` — both copies agree on the leak determination AND the main repo was restored to
  baseline. (Determination: leaks → emulate; no leak → invoke is safe.)
- `partial` — copies disagree, or the leak is conditional (only some writes leak).
- `broken` — a copy could not invoke the skill, or could not restore the repo.

---

## F3 — `merge=union` on a structured registry

**Determines:** whether a `merge=union` driver **silently** corrupts a *structured* registry
(multi-line blocks plus a trailing structural element) under concurrent appends — which
decides whether registries can be unioned or must be split + hand-resolved with a `tsc`
backstop. (Method §4 / FN-2 / FN-5 / FN-7.)

**Observe-recipe (in a THROWAWAY temp git repo — never the project repo):**
1. `git init` a temp repo. Add a structured registry file: several
   `export const X = { … } as const;` blocks followed by a trailing `type`/aggregate line.
   Set `merge=union` on that path via `.gitattributes`; commit.
2. Create 2–3 branches; on each, **append a new block** of the same shape.
3. Merge the branches into the base **in sequence**.
4. Inspect the merged file: is it still valid (e.g. every block keeps its closing
   `} as const;`), or did the 3-way union **splice** blocks (drop a closing token, nest one
   block inside another, drop a doc comment)? Capture the exact corruption shape if any, and
   note **which appender** first triggered it (often the 2nd or 3rd, never the 1st).

**Safety:** temp repo only; the project repo is never touched.

**Verdict rubric:**
- `clean` — both copies agree on the determination (corrupts / safe), with the merged bytes captured.
- `partial` — copies disagree.
- `broken` — git unavailable, or the temp repo could not be set up.

**Determination to record (example shape):** `"union silently splices on the 2nd appender — drops the prior block's '} as const;'; tsc-class error, git reports no conflict"`.

---

## F4 — Concurrent `worktree add`

**Determines:** whether near-simultaneous `git worktree add` locks/deadlocks `.git`, and
whether a bounded retry recovers — which decides whether lanes can be cut concurrently with a
simple retry, or must be serialized. (Method §8.)

**Observe-recipe (in a THROWAWAY temp git repo):**
1. `git init` a temp repo with at least one commit.
2. Launch ≥2 near-simultaneous `git worktree add` operations against it.
3. Observe whether a `.git` index/lock error occurs (capture the literal error).
4. Observe whether a bounded retry (2–3×, brief backoff) then succeeds.
5. Report: lock-occurred? retry-recovered? both worktrees created?

**Safety:** temp repo only.

**Verdict rubric:**
- `clean` — both copies agree AND the safe path holds: either no lock, or a lock that a
  bounded retry recovers from.
- `partial` — copies disagree, or recovery is flaky across the copies.
- `broken` — an **unrecoverable** deadlock (retry never recovers), or the temp repo setup
  failed. (An unrecoverable deadlock is a real escalation — the method's retry won't save it.)

---

## F5 — A declared dep's real API surface

**Determines:** whether an installed native dependency's **real export surface** matches what
a lane would assume (the ambient `.d.ts` shim) — catching cases like an API that moved to a
`/legacy` subpath, or a missing peer. (Method §8 / FN-1 / FN-5.)

**Input:** a dep name and the assumed surface (the exports/functions a lane uses). Sourced
from the skill's `dep` arg, else from a declared-but-maybe-uninstalled dep in
`{project-root}/package.json`. **If neither is available, mark `applicable: false`** —
this Fact is skipped, not escalated.

**Observe-recipe:**
1. Confirm the dep is installed (or install it in a throwaway scope).
2. Introspect the **real** exports: read the package's entry/`types`, or import it and
   enumerate the actual exported names and their shapes; read `peerDependencies`.
3. Compare the real surface to the assumed surface; list **matches** and **mismatches**
   (renamed/moved exports, subpath changes, missing peers).

**Safety:** prefer read-only introspection; if you install, do it in a throwaway scope, not
the project's `node_modules`.

**Verdict rubric:**
- `clean` — both copies agree AND the real surface matches the assumed surface.
- `partial` — copies disagree, or a partial match (some exports differ).
- `broken` — the dep could not be introspected (and a dep *was* supplied). If no dep was
  supplied at all, this is `applicable: false`, not `broken`.

---

## Extending the catalog (lever-6 and beyond)

To add a probe (e.g. the lever-6 *failure-probe* that injects a control fault and proves the
guard-rails scream): copy an entry's shape — a one-line **Determines**, an **observe-recipe**
of literal steps, a **safety** rule, and a **verdict rubric** that ties observed behavior to
`clean | partial | broken` under the same shared schema and the same ×2 confidence rule. The
host workflow needs no change; it iterates whatever Facts this catalog declares.
