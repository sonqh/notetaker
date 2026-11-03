# Proposal

## Why

The CLI from changes 001 to 004 works, but a meeting tool must start in one click. A terminal command is easy to forget when a call starts. Permission problems also stay hidden until a user reads a doctor report.

A menu bar app fixes both. It shows the state of the recording at a glance. It checks each permission by measured signal on first run. It offers to start notes when another app starts to use the microphone.

The app must work on a Mac that the user does not administer. Many work Macs are MDM managed, and the user has no admin rights. So the build, the install, and the signing stay at user level.

## What Changes

- Add `Notetaker.app`, a SwiftUI `MenuBarExtra` app with `LSUIElement` set. It is a thin front end over NotetakerKit, the same library that the CLI uses.
- Add `scripts/bundle.sh`. It builds the app bundle from SwiftPM with no Xcode project, signs it, and installs it to `~/Applications`. It uses no `sudo`.
- Sign the app with a stable self-signed identity from the user's login keychain, selected by name. Fall back to an ad hoc signature with a warning.
- Add a menu bar icon with three states: idle, recording, and warning.
- Add a popover with Start and Stop and an optional title. During a recording, it shows Me and Them meters, the elapsed time, and the last finalized lines. It also shows the current volatile line, a notes field, and warnings.
- Add a first-run onboarding window with a microphone step, a system audio step, and a speech model step. A step turns green only after the app measures signal.
- Detect a managed Mac that denies audio capture by policy. Tell the user to ask IT. Do not try to work around the policy.
- Add the mic-use prompt. When another app uses the microphone for about 3 seconds, post a notification that offers to start notes. The app never records on its own.
- Post a "Transcript saved" notification with Open, Show in Finder, and Copy for Claude Code.
- List the last 5 transcripts in the menu.
- Add settings for the output folder, the mic-use prompt, and the consent reminder.
- Remove the "A GUI or menu bar app" non-goal from `docs/spec.md`. The owner chose this direction in ADR-004.

## Capabilities

### New Capabilities

- `menu-bar-app`: the menu bar front end, its bundle and signing, the first-run onboarding, the mic-use prompt, the saved notification, the recent list, and the settings.

### Modified Capabilities

None. The app reuses the capture, watchdog, transcription, and output requirements from changes 001 to 004 without a change.

## Out of scope

- A notepad window. That is M5.
- A library of past meetings and search.
- Calendar data and EventKit.
- A start at login, a login item, or a LaunchAgent.
- A tap on one app only.
- Developer ID signing and notarization.
- Auto-update.
- Any install to `/Applications`, any driver, and any step that needs admin rights.
- Any summary or LLM feature, and any network call.

## Impact

- New executable target for the app in `Package.swift`. It depends only on NotetakerKit and Apple frameworks.
- New `scripts/bundle.sh` and new gate scripts `scripts/gates/005-*.sh`.
- View-model code that the tests can drive without audio hardware. This covers the icon state machine, the prompt debounce, and the bundle ID map. It also covers the recent list and the settings.
- `docs/spec.md` loses one non-goal. `README.md` gets install, signing, and managed Mac steps. `docs/findings.md` gets the app gate results.
- Refs: ADR-001, ADR-003, ADR-004, ADR-005, ADR-006, ADR-007.
