---
name: pwo-ui-smoke
description: "HELPER — not a step you run yourself; auto-invoked by steps 5 & 6 (pwo-run-wave / pwo-closeout). Smoke a rendered screen/journey against its mockup and return a verdict. Use when S3/S4 delegate a UI coherence smoke, or the user says \"smoke this screen\", \"ui coherence smoke\", \"pwo smoke\", or \"PWO S5\"."
---

# pwo-ui-smoke

Drive a **running** app through a journey and judge whether what renders **matches the
mockup** — the only counter-power to the UI blind spot, because a green CI renders no
pixels and a passing test double can hide a wrong number. You are **the delegated
sub-agent**, not an orchestrator: a thin caller (`pwo-run-wave` S3 at the end of a
screen-wave, `pwo-closeout` S4 for the final E2E smokes) hands you a journey and trusts
your verdict **without re-driving the app or re-reading your screenshots**. So your one
deliverable is a compact, reliable **StructuredOutput verdict** — never the transcript,
never the 40 screenshots. You fan out nothing and build nothing: you observe and report. And because you are a **leaf**, you emit
**no** next-step handoff — you return your verdict to your caller (`pwo-run-wave` / `pwo-closeout`),
which owns the pipeline relay.

**The non-negotiable:** judge **coherence against the mockup**, not merely the absence of
a crash — a screen that loads but renders the wrong layout, the wrong state, or the wrong
number is a FAIL. And **a claim with no captured screenshot behind it does not exist**:
report only what you observed on a real frame; a figure you did not actually see on screen
is `match:false`, not an optimistic guess. The orchestrator is thin and cannot catch a
verdict that bluffs.

## Resolution rules

- Bare paths and `{skill-root}` (e.g. `references/harness-targets.md`) resolve from this skill's installed directory.
- `{project-root}` → the project working directory.
- `pwo-ui-smoke` → the skill directory's basename.

## On Activation

Single-pass, headless, persona-free: no memlog, no resume, no customization. Resolve
inputs and run.

1. **Config.** Read `{smoke_harness}` — precedence: a value the caller passed you wins;
   else `smoke_harness` from `{project-root}/_bmad/config.yaml` (and `.user.yaml` if
   present); else default `expo-mcp`. Valid values: `expo-mcp | chrome-devtools | adb |
   none`. Degrade to `none` **ONLY when the configured value is `none`** — a **configured**
   harness that cannot be reached is `verdict: BLOCKED`, **never a silent degrade**: a
   jest-green PASS with zero pixels rendered would clear the exact gate this skill exists to
   hold, and the thin caller trusts the verdict without re-driving.
2. **Inputs** (the caller supplies these; ask the caller, don't invent):
   - **mockup(s)** — the reference screen(s)/region(s) from P2 (image paths or a mockup
     doc) that the render must match.
   - **journey** — the ordered steps to play (navigate here, enter X, expect screen Y with
     element Z) and the **expected live figures** to verify (e.g. `indemnité km = 127,20 €`).
   - **runtime state** — that the device/emulator/browser is already up (you are a
     *consumer* of running infra — see Gotchas).

## Bring up the harness

Read `references/harness-targets.md` and follow **only** the section for `{smoke_harness}`
— it carries that target's connect/drive/observe primitives. The shared discipline across
all targets: **connect to the already-running harness; never own shared infra.** Verify the
app is reachable, then drive it. If the harness genuinely cannot be reached, do not try to
stand up a second copy — emit `verdict: BLOCKED` with what you observed in `notes`.

## Play the journey and compare to the mockup

Walk the journey one screen at a time. For **every** screen the journey asserts:

1. **Capture a real frame** of the current screen and *read the image* — this is the
   ground truth; everything downstream derives from it.
2. **Coherence check vs the mockup.** Compare the captured frame to the corresponding
   mockup: are the expected elements present (cards, buttons, labels, list rows, the
   *empty vs populated* state), is the layout/hierarchy right, are there missing or
   spurious elements? Record each mismatch as a **delta** with a severity:
   `cosmetic` (spacing/shade) · `major` (wrong/missing element or state) · `blocking`
   (screen unusable / does not render the journey's subject).
3. **Verify the live figures.** For each expected figure, find it on the frame and compare
   **exactly** (currency, sign, decimals). A figure you cannot locate on a captured frame
   is `match:false`. These numbers are the C2 net — the on-device run is what catches a
   wrong value a green test-double let through.
4. **Note crashes/errors** — red screens, error toasts, blank renders (an HMR/WebSocket
   warning toast is not an app error — see Gotchas).

Keep the captured frames as evidence **files** and return their **paths** in the verdict;
do not inline images or a step-by-step transcript. If a journey step targets a screen
whose navigation entry does not exist yet (the wave hasn't composed its host), do **not**
FAIL — record it under `renderDeferred` and move on (this is expected mid-build; verify
reachability against the real router/nav, don't assume it).

## Emit the verdict

Your entire output is one object matching this schema (callers force it verbatim via the
Workflow `agent(..., {schema})` option — keep it stable; it is the contract):

```js
const VERDICT = { type:'object', additionalProperties:false, properties:{
  journey:        {type:'string'},                                   // what was played
  harness:        {type:'string', enum:['expo-mcp','chrome-devtools','adb','none']},
  verdict:        {type:'string', enum:['PASS','FAIL','BLOCKED']},
  screensVisited: {type:'array', items:{type:'string'}},
  deltas:         {type:'array', items:{type:'object', additionalProperties:false,
                    properties:{ screen:{type:'string'}, expected:{type:'string'},
                      observed:{type:'string'},
                      severity:{type:'string', enum:['cosmetic','major','blocking']} },
                    required:['screen','expected','observed','severity'] }},
  figures:        {type:'array', items:{type:'object', additionalProperties:false,
                    properties:{ label:{type:'string'}, expected:{type:'string'},
                      observed:{type:'string'}, match:{type:'boolean'} },
                    required:['label','expected','observed','match'] }},
  crashes:        {type:'array', items:{type:'string'}},
  renderDeferred: {type:'array', items:{type:'string'}},             // asserted but unreachable this wave
  uiBlindSpotCovered: {type:'boolean'},                              // false when harness=none
  evidencePaths:  {type:'array', items:{type:'string'}},            // screenshot files, NOT inlined
  notes:          {type:'string'} },
  required:['journey','harness','verdict','screensVisited','deltas','figures','crashes',
    'renderDeferred','uiBlindSpotCovered','evidencePaths','notes'] }
```

The verdict rule, applied mechanically: **PASS** iff zero `crashes`, zero `blocking`/`major`
deltas, every asserted figure `match:true`, **and every screen the journey asserts appears in
`screensVisited` with a captured frame behind it** (minus the `renderDeferred` ones) — a journey
you could not finish is a **FAIL** (or **BLOCKED** if the harness died mid-run), never a PASS on
the screens you did reach. Otherwise **FAIL** (cosmetic-only deltas still PASS but are reported).
**BLOCKED** iff the harness could not be brought up or reached (`harness ≠ none` — a configured
harness never degrades to `none`); say why in `notes`. Effort `high` suffices — reliability comes
from the capture-and-compare protocol above, not from deeper reasoning.

## Degrade when there is no UI harness

When `{smoke_harness} = none` — **configured** `none`, the only case that lands here (an
unreachable configured harness is `BLOCKED`, above) — there is no way to render pixels. Fall
back to the project's **jest / numeric sweep** as the only available signal: run it, set
`harness:'none'`, `uiBlindSpotCovered:false`, map green→`PASS` / red→`FAIL`, and put a
prominent `notes` warning that **the UI blind spot is uncovered — this verdict proves logic,
not render coherence.** A thin orchestrator must see that the mockup was never actually
compared (callers treat a screen-gate PASS with `harness:'none'` as BLOCKED).

## Gotchas (Android / Expo Go — they fire on situations you won't recognize)

- **`--go`, not plain `--android`.** A project with `expo-dev-client` still smokes in Expo
  Go via `npx expo start --go`; plain `--android` hunts a *development build* (`com.<app>`)
  and fails with "No development build installed."
- **Drive by screenshot, not resource-id.** `uiautomator dump` is unreliable on RN/Expo-Go
  (empty root) — capture `adb exec-out screencap -p > f.png`, *read the image*, and tap by
  coordinate.
- **Never `input keyevent 4` (BACK) to dismiss the keyboard** — it navigates back and exits
  the screen/app. Dismiss by tapping empty space.
- **Avoid taps within ~50 px of any screen edge** — Android's edge-swipe-back eats them, so
  a right-edge tab tap silently no-ops.
- **Bring-up is a consumer act, never an owner act.** The emulator, the adb server, and
  Metro (:8081) are **singletons owned by whoever launched them**. Do NOT boot a second
  emulator, run a second `expo start`, or `adb kill-server` — that wedges the shared infra.
  A stale Metro on :8081 or an `offline` emulator is the owner's to fix → report `BLOCKED`,
  don't restart it. An HMR / yellow "WebSocket connection" toast is not an app error.
