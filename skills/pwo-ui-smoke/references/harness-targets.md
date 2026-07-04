# Harness targets

Per-target connect / drive / observe primitives for the smoke. Follow **only** the section
named by `{smoke_harness}`. The rule that holds for every target: **you are a consumer of
already-running infra** — verify it is up and drive it; never boot a second copy or restart
a shared singleton. If the harness cannot be reached, emit `verdict: BLOCKED` (do not stand
up your own). Capture a real frame for every asserted screen and *read the image* — that
frame is the ground truth the verdict is built on.

## expo-mcp

Expo's device-automation MCP tools (`automation_tap`, `automation_find_view`,
`automation_take_screenshot`, `collect_app_logs`, `expo_router_sitemap`).

- **Connect.** These tools appear dynamically **only** when a dev server started with
  `EXPO_UNSTABLE_MCP_SERVER=1` is running **and** the app is connected. If they are not
  present in your toolset, the dev server isn't exposing them — that is the launcher's to
  fix; do not start your own. Fall back to the **adb** section's primitives to drive the
  same running emulator.
- **Drive.** `automation_find_view` by the RN `testID`, then `automation_tap`; read the
  router via `expo_router_sitemap` to plan the journey.
- **Observe.** `automation_take_screenshot` per asserted screen + *read the image*;
  `collect_app_logs` to catch runtime errors a screen swallows.
- **Known breakage.** On Windows these automation tools have shipped broken
  (`automation_tap` → `No quote function is defined (zx/quotes)`). When any of them errors,
  **fall back to the adb section** for tap/screenshot against the same emulator — that is
  the dependable path; re-check the MCP tools after an Expo update. (The *hosted* Expo MCP
  — `build_*`, `workflow_*`, `testflight_*` — builds/ships the APK; it does **not** drive a
  device, so it is not a smoke harness.)

## chrome-devtools

The Chrome DevTools MCP tools (`mcp__chrome-devtools__*`).

- **Connect.** `list_pages`/`select_page` to attach to the already-running page; if the app
  is not open, `navigate_page` to its dev URL. Don't spawn a second browser session.
- **Drive.** `take_snapshot` to get the a11y tree + element handles, then `click` / `fill`
  / `fill_form` by handle; `wait_for` text to settle between steps.
- **Observe.** `take_screenshot` per asserted screen + *read the image*;
  `list_console_messages` and `list_network_requests` to catch errors and failed loads.
- **Figures.** Read displayed values off the screenshot (and cross-check against the a11y
  snapshot text); compare exactly to the journey's expected figures.

## adb

The dependable Android path (Pixel-class emulator). The device is **1080×2400** on a
Pixel-7-class AVD; coordinates below are device pixels. Use the device id from
`adb devices` (e.g. `emulator-5554`).

- **Connect (consumer-only).** `adb devices` must show the emulator `device` (not
  `offline`, not absent). If it is offline or missing, the emulator/adb-server is the
  launcher's singleton — report `BLOCKED`; do **not** `adb kill-server`, boot a second
  emulator, or run a second `expo start`. (A project's dev launcher boots Pixel_7, sets
  `adb reverse tcp:8081 tcp:8081`, and opens the app via deep-link with `expo start --go` —
  that is owner work, already done before you run.)
- **Observe.** Screenshot with `adb -s <id> exec-out screencap -p > f.png` (binary-safe;
  prefer `exec-out` or `adb pull` over a plain shell `>` redirect, which CRLF-corrupts the
  PNG on Windows), then **read the image**. `uiautomator dump` is unreliable on RN/Expo-Go
  (empty root) — drive by reading the screenshot, not by the a11y XML.
- **Drive.** Locate the target element on the captured frame and tap its **center**:
  `adb -s <id> shell input tap X Y`. Type with `adb -s <id> shell input text "..."`.
  Two traps that silently no-op or break the journey: **never `input keyevent 4`** (BACK —
  it exits the screen; dismiss the keyboard by tapping empty space), and **keep taps ≥ ~50
  px from every screen edge** (edge-swipe-back eats edge taps).
- **Figures.** Read the displayed numbers off the screenshot and compare exactly
  (currency, sign, decimals) to the journey's expected figures.
- **Teardown.** None — you started nothing. Leave the emulator, adb server, and Metro
  running for the owner.

## none

No UI harness is **configured** (this section is never a degrade target — a configured harness
that cannot be reached is `BLOCKED`, not `none`). There is no way to render pixels, so the UI
blind spot **cannot** be covered.

- Run the project's **jest suite and/or the numeric sweep** as the only available signal.
- Build the verdict with `harness:'none'`, `uiBlindSpotCovered:false`, green→`PASS` /
  red→`FAIL`, empty `evidencePaths`, and a prominent `notes` warning that the mockup was
  never compared — **this verdict proves logic, not render coherence.** The thin
  orchestrator must be able to see that no pixels were checked.
