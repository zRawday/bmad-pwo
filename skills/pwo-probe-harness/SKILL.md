---
name: pwo-probe-harness
description: Empirically probe what the parallel-build harness allows. Use when the user says "probe the harness", "run the harness probe", "characterize the harness", "pwo probe", or before planning parallel waves (PWO S0).
---

# pwo-probe-harness

Establish — by **executing**, never by reasoning — what this harness actually allows, so the
wave planner (`pwo-plan-waves`, S1) builds the parallelization mechanism on fact. The output
is a **harness-profile** artifact: a `clean | partial | broken` verdict per harness Fact plus
the raw observations behind it. S1 is the consumer and it sets the bar — it refuses to freeze
the mechanism unless every applicable Fact is `clean` or its escalation has been resolved.
This skill is **project-agnostic, headless, and persona-free**: it fans out probe subagents,
folds their verdicts, and either writes a clean profile or escalates. It plans and builds
nothing else.

**The non-negotiable:** output is 100% factual. Run **two identical copies per probe** —
agreement is the only source of confidence. Any applicable Fact that lands `≠ clean` is
**escalated to the human**, because the whole pipeline mechanism downstream depends on it.

## Resolution rules

- Bare paths and `{skill-root}` (e.g. `references/probe-catalog.md`) resolve from this skill's installed directory.
- `{project-root}` → the project working directory.
- `{skill-name}` → the skill directory's basename.

## On Activation

1. Load config from `{project-root}/_bmad/config.yaml` (and `.user.yaml` if present), using
   defaults for anything missing — this skill must run on a bare repo. Resolve:
   `{project-name}` (else the repo folder name), `{output_folder}` (else `{project-root}/_bmad-output`),
   and the **main branch** (config `main_branch`, else `git rev-parse --abbrev-ref HEAD`).
2. Read the inputs (all optional, headless-safe defaults): a **dep** name + its assumed export
   surface for F5 (else a declared-but-maybe-uninstalled dep from `{project-root}/package.json`;
   else F5 is `applicable:false`); **copies** per probe (default 2); a **sandbox** dir for the
   temp-repo probes (default the OS temp dir); the **output path** (default
   `{output_folder}/pwo/harness-profile.md`).

No memlog, no resume, no customization: this is a single-pass probe, not a revisable artifact.

## Run the probes

Read `references/probe-catalog.md` — it holds, per Fact, the observe-recipe, the safety rule,
and the verdict rubric. **Author one probe subagent prompt per Fact from its catalog entry**,
and run **each Fact twice** as **workflow subagents via the Workflow tool**. Facts F1 and F2
characterize *a workflow subagent itself* (its toolset, whether the real Skill leaks), so they
MUST run inside a workflow — a top-level Agent would measure the wrong thing. Your invocation
of this skill is the Workflow opt-in.

Every probe prompt carries the catalog's non-negotiable verbatim in spirit — *report what you
observe by executing; never guess; a determination without captured evidence is `broken`, not a
guess* — and forces the shared StructuredOutput schema so the return is machine-checkable.

The fan-out shape (author the FACTS array and the prompts from the catalog; do not invent a
different schema):

```js
export const meta = { name: 'pwo-harness-probe', description: 'Probe harness Facts; 2 copies each; ground-truth verdicts', phases: [{ title: 'Probe' }] }
const FACTS = [ /* one entry per APPLICABLE Fact, authored from references/probe-catalog.md: { id, recipe, safety, rubric } */ ]
const COPIES = 2
const SCHEMA = { type:'object', additionalProperties:false, properties:{
  factId:{type:'string'}, probeRan:{type:'boolean'}, applicable:{type:'boolean'},
  observations:{type:'string'}, determination:{type:'string'},
  verdict:{type:'string', enum:['clean','partial','broken']}, verdictReason:{type:'string'},
  safetyRestored:{type:'boolean'}, notes:{type:'string'} },
  required:['factId','probeRan','applicable','observations','determination','verdict','verdictReason','safetyRestored','notes'] }
const probePrompt = (f, copy) => `You are probe copy ${copy} for Fact ${f.id}.
REPORT WHAT YOU OBSERVE BY EXECUTING. NEVER GUESS. A determination with no captured command output /
file content / tool list / error string does not exist — that is verdict "broken", not a guess.
Execute this recipe exactly, including any restore/teardown; do not skip a step:
${f.recipe}
Safety: ${f.safety}
Return ONLY the schema. observations = raw ground truth; determination = derived ONLY from it;
verdict per this rubric: ${f.rubric}`
phase('Probe')
const runs = (await parallel(FACTS.flatMap(f =>
  Array.from({length: COPIES}, (_, c) => () =>
    agent(probePrompt(f, c+1), { label:`probe:${f.id}.${c+1}`, phase:'Probe', schema:SCHEMA, effort:'high' }))
))).filter(Boolean)
return { runs }   // COPIES × FACTS verdict objects
```

## Fold each Fact, then assemble the profile

Group the returned `runs` by `factId` and fold the two copies into one Fact verdict (the rule
is in the catalog, "The universal confidence rule"):

- both copies ran and **agree** on the determination, matching the Fact's `clean` rubric → **clean**;
- the copies **disagree**, or a copy is conditional/incomplete → **partial**;
- a copy could not execute the probe → **broken**.

Disagreement is never `clean`, whatever a copy claimed — a non-deterministic harness is itself
a reason not to trust the Fact. A Fact marked `applicable:false` (e.g. F5 with no dep) is
recorded as such and never escalated.

Write the artifact from `assets/harness-profile-template.md` to the resolved output path:
the verdict summary table, **both copies' raw observations** per Fact, the S1 implication, and
an escalation block for each non-clean Fact. Verify F2/F3/F4 reported `safetyRestored:true`;
if a real-repo probe did not restore, say so prominently — that is itself an escalation.

## Escalate any non-clean Fact

If every applicable Fact is `clean`, the profile is `overall: clean` — report the path and
stop. Otherwise, for each non-clean Fact, **escalate to the human** (talk in
`{communication_language}`): state what was observed (the disagreement, or the broken/partial
state), why it blocks S1, and present the options — **re-probe ×2 · investigate manually ·
accept with documented risk · adapt the mechanism**. Use AskUserQuestion. Record the human's
decision in the profile's escalation block. Do not silently downgrade or paper over a
non-clean Fact; S1 keys off this verdict.

## Dependencies and degradation

- **The Workflow tool is required** for F1/F2 (they measure workflow-subagent behavior). If it
  is unavailable, mark F1/F2 `broken` and escalate — never substitute a top-level Agent, which
  would characterize the wrong harness.
- **git with worktree support** is required for F2/F3/F4; F3/F4 run in a throwaway temp repo so
  the project repo is never touched, and F2 captures-then-restores the main repo.
- **F5 degrades to `applicable:false`** when no dep is supplied or declared — it is skipped, not failed.

## Headless output

When invoked headless, skip the interactive escalation: still write the profile, mark each
non-clean Fact's escalation block `pending-human`, and return JSON only —
`{ "status": "complete" | "blocked", "overall": "clean" | "needs-attention", "profile": "<path>", "nonClean": ["F?", …] }`
(`"blocked"` with a one-line `reason` if a probe could not run at all).
