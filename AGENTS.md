---
alwaysApply: true
---

# pebble-tap-tap-timers

Tap Tap Timers is a watchapp for Pebble Time 2 (emery): several countdown timers started from presets, with configurable vibration, a pre-end warning, shake-to-dismiss and full button control. Written in C with the Pebble SDK; the phone part is PebbleKit JS. Current version: 0.1.0, not yet released. Change history is in `CHANGELOG.md`.

Unofficial remake of Two Tap Timers by Arney: https://apps.repebble.com/two-tap-timers_c0609699094043b68b8bcfed. Written from scratch: do not copy or reproduce the original's code; its license is unknown.

Talk to the user in Ukrainian. Code, comments, variable names, UI strings, the settings page, README and commit messages are in English.

## Working rules

- Plan first: for any non-trivial task, present a plan and wait for approval, unless the user says otherwise.
- Implement every item of the approved plan. If an item is dropped or changed, say so explicitly in the report.
- In reports, separate what was verified (with evidence: test output, build output, a screenshot, a verbatim quote from the docs with its URL) from what was not. Never write "verified", "verbatim" or "working" without evidence.
- Do not install or update packages or tools (pebble-tool, SDKs, `pebble package install`, npm) without permission. The user pins the pebble-tool version.
- Do not use `pebble new-project --ai`: this file already provides the agent instructions.

## Goal and scope

Reliable timers with vibration that can be configured. Nothing more.

Never do:
- cloud, accounts, syncing with services;
- resetting running timers by shaking or any other implicit gesture.

Not in v1, only on explicit request: App Glance, touch, timer names, interval mode (rounds), presets in seconds, Round 2. Quick Launch is a system setting of Pebble; there is nothing to code.

## Features (v1)

- Up to 4 concurrent timers. Start from a preset or from a custom time in the editor; pause, resume, reset.
- Clock at the top (StatusBarLayer).
- Each timer shows remaining time and end time, e.g. "7:32 → 17:49". The end time follows the system 12h/24h setting.
- Pre-end warning: one short pulse of 50–80 ms. Setting: off / 30 s / 1 min (default) / 2 min.
- Vibration settings (the SDK only controls pulse and pause durations, not motor strength):
  - alarm pattern: short / long / double / escalating;
  - separate warning pattern;
  - ring duration without reaction: 15 s / 30 s / 1 min / until dismissed. After it, silence, and the alarm screen shows overtime "+m:ss";
  - vibrate during Quiet Time: yes/no.
- Shaking dismisses the alarm, on the alarm screen only: `accel_tap_service`, trigger = two taps within ~600 ms. Taps during vibration and ~150 ms after it are ignored, because the motor itself produces taps. After dismissal, a short confirmation pulse. Setting on/off. Tap subscription only while the alarm screen is open. Threshold and windows are tuned on the real watch.
- Buttons, main screen:
  - Up/Down – select;
  - Select on a preset – start;
  - Select on an active timer – pause/resume;
  - long Select on an active timer – reset with a confirmation pulse;
  - long Select on "Custom" – time editor (Up/Down change the value, Select moves to the next field);
  - Back – exit to the watchface via `exit_reason_set(APP_EXIT_ACTION_PERFORMED_SUCCESSFULLY)`.
- Buttons, alarm screen:
  - Select – dismiss;
  - Up – +1 min;
  - Down – +5 min;
  - long Select – restart the same timer;
  - Back – dismiss and exit to the watchface;
  - shake – dismiss.
- The backlight is on during the alarm only while it vibrates (at most 1 min), not all the way until dismissal.
- Double-press guard of 700 ms on actions that start, reset, restart or extend a timer.
- Presets (8 of them) and all settings are edited on the settings page on the phone. The page is English only.
- Persistence: closing the app or rebooting the watch does not lose timers.

## Architecture

Layout follows the Modular App Architecture guide: wscript compiles every `.c` under `src/c/`, including subfolders; each module is an `.h`/`.c` pair, headers start with `#pragma once`, module data is `static`, each window is its own module with a push function, and `main.c` only initialises and deinitialises modules. The approved v1 plan defines the exact files, keys and protocol; after implementation, describe the actual state here.

### Watch (`src/c/`)

- `main.c` – entry point: launch reason, state restore, cancelling own wakeups, initialising modules and windows.
- `modules/schedule.c` – pure event scheduling core, without `pebble.h`. Compiled both by the SDK build and by the host compiler for tests.
- `modules/timers.c` – timer state and transitions (idle / running / paused / ringing / overtime).
- `modules/storage.c` – persistent storage, schema number, migrations.
- `modules/settings.c` – settings and presets, receiving them from the phone.
- `modules/vibes.c` – vibration patterns, Quiet Time, tap ignore windows.
- `windows/main_window.c`, `windows/editor_window.c`, `windows/alarm_window.c`.

### Phone (`src/pkjs/`)

- `index.js` – PebbleKit JS events, opening the settings page, sending settings to the watch.
- Settings page: either Clay (`@rebble/clay`, the officially recommended way, `src/pkjs/config.js`) or a custom data-URI page as in Spool. The user decides; do not install the package before the decision.

### Settings on the watch

All settings and presets are one struct, saved with `persist_write_data` under a fixed key after every update from the phone (the pattern from the App Configuration guide). The struct must fit in 256 bytes. Missing or invalid fields fall back to defaults.

## Event scheduling (critical)

Wakeup API limits (Wakeup guide; check the `.md` version before writing code):
- at most 8 scheduled wakeups per app; the ninth returns `E_OUT_OF_RESOURCES` (-7);
- each wakeup takes a ±1 min window in which no app on the watch can schedule another one; a clash returns `E_RANGE` (-8);
- not within 30 s of now; a time in the past returns `E_INVALID_ARGUMENT` (-4); `E_INTERNAL` (-3) is also possible;
- a non-negative value is a `WakeupId`; the on-watch behaviour of `notify_if_missed` is not documented and is checked on the real watch.

Required design:
- On every launch: restore state from storage, cancel own wakeups.
- While the app is open, all events (alarms, warnings) are driven by `app_timer`, no wakeups are scheduled. In addition, every tick checks events against absolute time, so a change of system time does not break the countdown. `wakeup_service_subscribe` stays subscribed as a safety net.
- On exit: save state first, then schedule only the nearest event as a wakeup; persist its `WakeupId`.
- On launch with `APP_LAUNCH_WAKEUP`: handle every event that is due or due within the next minute, then schedule the next nearest one.
- If `wakeup_schedule` returns an error, search for the nearest free earlier slot by stepping through candidates. An alarm has priority over a warning: a conflicting warning is shifted or skipped.
- All logic for picking events, merging them and choosing a slot lives in `modules/schedule.c` with host tests (`cc` + a minimal runner in `tests/`). Run the tests before every commit that touches this module. Error codes in the tests are modelled exactly as -3, -4, -7, -8.

## Storage on the watch

- The schema number is its own key; the first schema version is 1.
- Running timers are stored with their absolute end time; paused timers with their remaining time. The display is always recomputed from absolute time, never from a tick counter.
- One value is at most 256 bytes (`PERSIST_DATA_MAX_LENGTH`); check the total per-app limit in the Persistent Storage guide.
- Migrations in a safe order: new keys first, then the schema number, then deleting old keys. An interrupted migration completes on the next launch.
- Write on state change, not on every tick.

## Technical constraints and verified facts

- The "→" glyph may be missing from system fonts: check it in the emulator; if it is missing, propose a replacement. The UI is in English, so no custom font for Cyrillic is needed (unlike Spool).
- Power use (Conserving Battery Life guide): second ticks only while a countdown is visible on screen, otherwise minute ticks; the Vibes API used sparingly and with short patterns; the backlight only for short periods; the accelerometer only via tap events, no sampling.
- `quiet_time_is_active()`: there is no page for it in the reference; check that it exists and its signature in `pebble.h` of the installed SDK.
- The build reports that `package.json` has no `usesPermissions`; this is expected: the app uses neither network nor location.
- A clean build (`pebble clean`) changes the resource timestamp in the `.pbw`, so a `.pbw` from the same code differs byte for byte.
- The phone is a Samsung S26 with Android. The Pebble app on Android runs PebbleKit JS in a hidden WebView. Before changing phone code, read the PebbleKit JS guide (`using-pebblekit-js.md`).
- Verified in Spool on the Samsung S26: a settings page as a data URI opens in the Pebble app. Clay has not been tested on this phone yet.
- The emulator does not test the page inside the Pebble app: `pebble emu-app-config` opens it in the computer's browser.
- The emulator cannot check vibration feel, shake thresholds, false taps from the motor, wakeups on the real watch, or `notify_if_missed` behaviour. In reports, mark all of these as "needs real-watch check".

## Platform and configuration

- Target platform is `emery` only (Pebble Time 2). `pebble.targetPlatforms` in `package.json` must contain only `emery`.
- This is a watchapp, not a watchface: `pebble.watchapp.watchface` must be `false`.
- The app is configurable: `pebble.capabilities` must contain `configurable`.
- `pebble.displayName` is `Tap Tap Timers`. The top-level `name` is `pebble-tap-tap-timers`, matching the repository. `author` is `rootatoskr`.
- `pebble.uuid` is new, generated by `pebble new-project`; never edit it by hand or take it from other projects.
- `version`: the docs require the `major.minor.0` format; check what the build accepts.
- `pebble.messageKeys` lists every settings and preset key; an undeclared string key is not transmitted.
- Launcher icon: `resources/icon.png` (25×25, `"menuIcon": true`).

## Project structure

```
src/c/ - watch app code (C)
src/c/modules/ - modules, including the pure scheduling core
src/c/windows/ - windows
src/pkjs/ - phone code (PebbleKit JS)
resources/ - launcher icon
tests/ - host tests for the scheduling core (C, outside src/c/ so they stay out of the app build)
package.json - metadata, platforms, AppMessage keys, resources
CHANGELOG.md - version history
AGENTS.md, CLAUDE.md - instructions for the coding agent
screenshots/ - .pbw and store materials, not in the repository
```

## Commands

```bash
# build
pebble build
# clean the build
pebble clean
# host tests for the scheduling core (the v1 plan defines the exact command)
make -C tests
# install to the Time 2 emulator
pebble install --emulator emery
# open the app's settings page in the browser
pebble emu-app-config --emulator emery
# screenshot
pebble screenshot --scale 6 --no-open screenshot.png
# press a button
pebble emu-button click select
# accelerometer tap
pebble emu-tap --direction x+
# set emulator time (HH:MM:SS or Unix timestamp)
pebble emu-set-time 17:48:30
# 12h/24h time format
pebble emu-time-format --format 24h
# APP_LOG logs, until Ctrl-C
pebble logs --emulator emery
```

For help on any command, add `--help`. Do not invent options that are not in `--help`.

## Checks after every change

Do not stop at a successful build.

1. `pebble build` with no warnings, plus the host tests for the scheduling core.
2. `pebble install --emulator emery --logs` – wait for the app to start, check the logs, stop with Ctrl-C.
3. `pebble emu-app-config --emulator emery` – change presets and settings, save; check that the watch received and stored them.
4. `pebble screenshot --scale 6 --no-open screenshot.png` – look at the screenshot.
5. Starting a timer: `pebble emu-button click select` on a preset, screenshot: remaining time and "→ end time".
6. Pause and reset: `pebble emu-button click select`, then `pebble emu-button click select --duration 1000`, screenshots.
7. Editor: long Select on "Custom", change fields with Up/Down/Select, screenshot.
8. Alarm: a short timer or `emu-set-time`; check the alarm screen, +1/+5 min, restart, "+m:ss" after the ring duration.
9. Shake: `pebble emu-tap` twice less than 600 ms apart on the alarm screen dismisses it; on the main screen it does nothing.
10. Persistence: close the app (`pebble emu-button click back --duration 2000`), open it again – timers, pause state and settings must remain, with the correct remaining time.
11. Time format: `pebble emu-time-format --format 12h`, screenshot.

Test scenarios:
- 4 timers at once, two ending within the same minute;
- a warning that coincides with another timer's end;
- a paused timer while the app is closed;
- closing the app less than 30 s before a timer ends;
- a system time change during a countdown.

After checking, ask yourself whether the screenshot matches the task, and fix any mismatch.

Mark temporary edits for simulating failures or measuring with `HACK` on every line and remove them after the check: `grep -c HACK` over the code must give 0, and `git status` must be clean.

## Controlling the emulator

- `click` – press and release; `--duration` for a long press.
- `push` / `release` – hold / release.
- `--repeat N` and `--interval ms` – repeated presses.
- Buttons: `back`, `up`, `select`, `down`.
- `emu-tap --direction` – `x+`, `x-`, `y+`, `y-`, `z+`, `z-`. Tests logic only, not thresholds.

If the emulator behaves oddly (screenshot of another app, install hangs): `pebble kill`, then `pebble wipe`, then install again.

## Installing on the watch

Only when the user asks, by any method (`--cloudpebble`, `--phone`):

```bash
pebble install --cloudpebble
```

Do not keep `pebble logs` or `--logs` open on the real watch longer than needed: the log stream keeps Bluetooth active and drains both the watch and the phone.

## Release

- The version is in `package.json` (`version`), together with a section in `CHANGELOG.md`.
- Tag `vX.Y.Z` (annotated, "Version X.Y.Z") goes on the commit the `.pbw` was built from, after the user has checked it on the real watch.
- The `.pbw` (`screenshots/tap-tap-timers-X.Y.Z.pbw`) and store materials (screenshots, description, release notes, icons, banner) live in `screenshots/`, which is in `.gitignore`.
- Store materials: screenshots exactly 200×228 (GIF allowed), banner 720×320 PNG/JPEG, icons 80 and 144. The store description and README start with the line "Unofficial remake of Two Tap Timers by Arney."
- The user publishes manually via the Developer Dashboard (developer.repebble.com/dashboard, New Release). Only the user changes visibility (Listed / Unlisted / Hidden) in the dashboard.

## Git

- After every working step checked in the emulator, make a separate logical commit with a short English description. Commit nothing if the build or tests fail.
- `AGENTS.md` and `CLAUDE.md` are tracked in the repository. Commits may add or change them, and commit messages may name them (e.g. "Update AGENTS.md").
- Apart from these two files, no mentions of Claude, Anthropic or AI assistants: not in commit messages, code, comments, README, CHANGELOG or other project files. No signatures and no trailers (Co-Authored-By or others). Do not describe any work as AI-generated.
- Author and committer are the GitHub no-reply address `rootatoskr <129084982+rootatoskr@users.noreply.github.com>`, in the repository's local config. Never use the user's personal e-mail.
- After a commit, show `git log -1 --format='%an <%ae>%n%n%B'`.
- Push only on the user's command, and never with force.

## Documentation

- Index of all pages: https://developer.repebble.com/llms.txt
- Every page has a Markdown version: replace the trailing `/` with `.md`. Read the `.md` version before using an API.
- Before using a function, check it in the reference or in `pebble.h` of the installed SDK, not from memory.

Useful sections:
- Wakeup (guide): https://developer.repebble.com/guides/events-and-services/wakeups.md
- Wakeup (API): https://developer.repebble.com/docs/c/Foundation/Wakeup.md
- Timer (app_timer): https://developer.repebble.com/docs/c/Foundation/Timer.md
- Wall Time: https://developer.repebble.com/docs/c/Foundation/Wall_Time.md
- Vibes: https://developer.repebble.com/docs/c/User_Interface/Vibes.md
- Light: https://developer.repebble.com/docs/c/User_Interface/Light.md
- Accelerometer (guide): https://developer.repebble.com/guides/events-and-services/accelerometer.md
- AccelerometerService: https://developer.repebble.com/docs/c/Foundation/Event_Service/AccelerometerService.md
- Persistent Storage: https://developer.repebble.com/guides/events-and-services/persistent-storage.md
- Storage (API): https://developer.repebble.com/docs/c/Foundation/Storage.md
- StatusBarLayer: https://developer.repebble.com/docs/c/User_Interface/Layers/StatusBarLayer.md
- Buttons: https://developer.repebble.com/guides/events-and-services/buttons.md
- App Exit Reason: https://developer.repebble.com/guides/user-interfaces/app-exit-reason.md
- App Configuration (Clay): https://developer.repebble.com/guides/user-interfaces/app-configuration.md
- App Configuration (manual setup): https://developer.repebble.com/guides/user-interfaces/app-configuration-static.md
- PebbleKit JS: https://developer.repebble.com/guides/communication/using-pebblekit-js.md
- AppMessage: https://developer.repebble.com/docs/c/Foundation/AppMessage.md
- Modular App Architecture: https://developer.repebble.com/guides/best-practices/modular-app-architecture.md
- Conserving Battery Life: https://developer.repebble.com/guides/best-practices/conserving-battery-life.md
- App Metadata: https://developer.repebble.com/guides/tools-and-resources/app-metadata.md
- pebble tool: https://developer.repebble.com/guides/tools-and-resources/pebble-tool.md
- Preparing a Submission: https://developer.repebble.com/guides/appstore-publishing/preparing-a-submission.md
- Publishing an App: https://developer.repebble.com/guides/appstore-publishing/publishing-an-app.md
