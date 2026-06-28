---
title: Method + Skills Maturation Report
produced_by: pwo-closeout (S4)
consumed_by: the BMad Builder (bmad-workflow-builder, Edit intent) — filtered by fix-target
project: {project-name}
ran: {timestamp}
sources:                                  # this report POINTS at these; it does not duplicate them
  - parallel-build run log (Field Notes FN-1…FN-N)
  - wave run-records (S3) — critic flags · integration manual-resolutions · review escalations
  - deferred-work
  - this closeout's §A campaign + §B consolidation
absorbs: the standalone "method retro" (no separate document — it lives here)
entry_count: {n}
---

# Method + Skills Maturation Report — {project-name}

> The suite's **memory about itself** — the one genuine memory concern of PWO, and the artifact that
> makes it **self-improving**. It aggregates the run's frictions **by pointing** at the evidence (FN ids,
> commits, deferred items), **never re-narrating** them, into **one entry per problem**. It is
> **reinjectable into the BMad Builder** filtered by `fix-target`: each `fix-target = Sx` entry is a
> ready-made Edit brief that hardens the skill that produced it. It **absorbs the former standalone
> "method retro"** (the headline below) so nothing proliferates.

## Headline (the absorbed method-retro)

- **Verdict:** {method validated at this run's scale — e.g. "0 main corruptions over N lanes, 0→T tests
  additive, ~Kx speedup" — or the honest reservation if not}.
- **Highest-leverage open items:** {the 1–3 `status: open` entries that would most improve the next run}.
- **What is now closed by a lever:** {the recurring frictions a lever already eliminates — point at the entries}.

## Entries — one per problem

> Dedup ruthlessly: a friction that recurred every wave is **one** entry (`recurrence: every wave`), not
> N. `trace` is a **pointer**, not the story. `category` ∈ {`intrinsic-parallelization` · `stack-specific`
> · `harness`}. `status` ∈ {`lever 1…10` · `open`}. `fix-target` ∈ {`method doc §X` · `skill Sx` ·
> `Phase 0` · `tooling` · `P2`} — the routing key for reinjection.

| # | Problem | trace | recurrence | severity | root cause | category | status | fix-target | action |
|---|---------|-------|------------|----------|-----------|----------|--------|-----------|--------|
| 1 | {short name} | {FN-x · sha · deferred item} | {every wave / once @ Wn / N lanes} | {silent-corruption / money / friction / cosmetic} | {the mechanism, 1 line} | {intrinsic-parallelization} | {lever N / open} | {skill Sx / Phase 0 / method §X / tooling / P2} | {the concrete next step} |
| 2 | … | … | … | … | … | … | … | … | … |

<!-- Illustrative rows from the first full-scale run (Gesto) — they teach the schema; replace with THIS
     project's. Notice each `trace` points; none re-narrates. -->

| # | Problem | trace | recurrence | severity | root cause | category | status | fix-target | action |
|---|---------|-------|------------|----------|-----------|----------|--------|-----------|--------|
| e1 | `merge=union` silently corrupts the structured test-id registry | FN-2/4/5/7 | every wave (silent even on a "clean" rebase) | silent-corruption-risk | union nests a structured block inside the prior one, dropping its `} as const;`; only tsc catches it | intrinsic-parallelization | lever 1 (closed) | Phase 0 / skill S2 | S2 ships `test-ids/<screen>.ts` + barrel so each lane writes only its own file — zero union. Confirm S2 enforces it. |
| e2 | the in-memory test double is FK-blind | FN-6/FN-7 · removeEtape fix sha | 2 waves (1 shipped-orphan, 1 caught at gate) | data-correctness (on-device FK throw) | the double enforces no FKs → a wrong delete order surfaces only as an orphan, never the on-device throw | intrinsic-parallelization + stack-specific | lever 2 (open) | tooling + skill S3 | make the double FK-aware (interpret REFERENCES + throw) AND keep the gate's delete-path audit when a lane adds the first referencing rows. |
| e3 | the same figure re-implemented across isolated lanes | FN-4/FN-6 | 3 lanes | divergence-risk | isolation gives no shared neutral home, so each lane re-derives the formula | intrinsic-parallelization | lever 5 + lever 10 (closed this run) | skill S3 + skill S4 | S3's critic flags it + a temp parity check; S4's §B consolidated it into one helper + a permanent parity test. |
| e4 | a hoisted forward-seam is positionally wrong for its eventual consumer | FN-7 / A1 | once @ W5 (caught) | silent-corruption-risk | a seam comment encodes the EARLIER wave's ordering (here FK-unsafe) and the dev follows shipped code over prose | intrinsic-parallelization | lever 4 (closed) | skill S3 | S3's mandatory gate fact-extraction reads the real seam body + re-validates position; bake an authoritative amendment. Keep it. |
| e5 | a declared native dep's real export surface differs from the assumed one | FN-1/FN-7 | 2 deps | integration-friction | the installed API can move (a `/legacy` subpath) vs what the lane's shim assumed | stack-specific + harness | lever 3 (refined) | skill S2 + method doc §8 | pre-provision on main AND pre-fetch the dep's real API surface from its docs before the install (pre-empts the mismatch). |

## Reinject into the Builder (the self-improving loop)

> **Filter the entries by `fix-target`.** Each group is a work package for the next iteration; the
> `skill Sx` groups are ready Edit briefs for `bmad-workflow-builder`.

- **→ skill S0 / S1 / S2 / S3 / S4 / S5:** {the entries targeting each skill — reinject into the Builder
  (Edit intent on that skill) to harden it. These turn this run's reactive frictions into the next run's
  proactive guard-rails.}
- **→ Phase 0:** {entries that sharpen S2's guard/seam/dep/test-id list}.
- **→ method doc §X:** {entries that mature the method or its pre-conditions}.
- **→ tooling:** {entries needing infra — e.g. an FK-aware double, a custom merge-driver}.
- **→ P2:** {entries about the mockups / smoke harness coverage}.

Each project's closeout thus **hardens the skills that produced it** — the module gets better at its own
job every run.

## Open retro questions to fold in (if they sharpen a lever)

- {where the human's cognitive load sat, distinct from the orchestrator context cost}
- {whether the C1 manual resolution was truly the worst pain, or another (dep-shim / smoke / session discipline)}
- {any moment the "main stays healthy" confidence wavered — feeds the failure-probe (lever 6)}
