# Design

## Context

Changes 001 to 004 build NotetakerKit and the `notetaker` CLI. NotetakerKit holds capture, the watchdog, transcription, the notes, and the Markdown writer. The CLI is a thin front end over it.

This change adds a second thin front end: a menu bar app. The app must build with SwiftPM alone, because the repo has no Xcode project. It must install and run with no admin rights, because the Mac can be MDM managed. A management profile can also deny audio capture by policy (PPPC), and the app cannot change that.

TCC ties each grant to the code signature of the app. An ad hoc signature changes on every build, so grants can reset after each rebuild.

## Goals / Non-Goals

**Goals:**

- One click to start and stop a recording from the menu bar.
- A first-run check that proves each permission by measured signal.
- An offer to start notes when a call app starts to use the microphone.
- A build, install, and signing flow that needs no admin rights.
- The same file format and the same library as the CLI.

**Non-Goals:**

- A notepad window, a library, search, or calendar data.
- A start at login, a login item, or a LaunchAgent.
- A tap on one app only.
- Developer ID signing, notarization, and auto-update.
- Any summary feature or network call.

## Decisions

- Build the bundle with `scripts/bundle.sh`: `swift build -c release`, then copy the binary to `Contents/MacOS`, write `Info.plist`, sign, and copy to `~/Applications`. An Xcode project would add a second build system and a file that drifts from `Package.swift`. Refs: REQ-APP-001, ADR-001, ADR-005
- Install to `~/Applications`, never `/Applications`. A standard user can write there with no `sudo`. macOS finds apps in that folder as it finds apps in `/Applications`. Refs: REQ-APP-001, ADR-005
- Sign with a stable self-signed code signing identity in the login keychain. The user creates it once in Keychain Access by the steps in `README.md`. The script reads its name from `NOTETAKER_SIGN_IDENTITY`. A Developer ID would need a paid account and is out of scope. Refs: REQ-APP-002, ADR-005
- With no identity set, sign ad hoc and print a warning. This keeps a first build easy and makes the cost clear. The script never creates a certificate, because that changes the keychain trust settings and can need admin rights. Refs: REQ-APP-002, ADR-005
- Put the app in a new executable target that depends only on NotetakerKit. All logic that the tests must reach lives in view models with no SwiftUI import. These are the icon state machine, the prompt debounce, the bundle ID map, the recent list, and the settings. Refs: REQ-APP-003, REQ-APP-012, ADR-004
- Derive the icon state from one pure function of the session state and the two watchdog states. Warning wins over recording. This keeps the state testable with synthetic watchdog input. Refs: REQ-APP-003, ADR-003
- Feed the popover from the same session events that the CLI prints: meter levels, finalized lines, the volatile line, and warnings. The app throttles meter updates to the main actor at about 15 Hz. The audio callback never touches the UI. Refs: REQ-APP-004, ADR-003
- Write notes through the NotetakerKit notes API that the CLI uses. The app adds no format of its own. Refs: REQ-APP-005, REQ-APP-012, ADR-006
- Reuse the doctor checks from NotetakerKit for onboarding. The microphone step uses a live Me meter. The system audio step runs the Tink self-test with `afplay` in a separate process. The model step reads the asset state and shows the OS download progress. Refs: REQ-APP-006, ADR-002, ADR-003
- Open System Settings panes with their `x-apple.systempreferences:` URLs through `NSWorkspace`. The button does nothing else. Refs: REQ-APP-006, ADR-005
- Treat a step that stays silent after a grant and after a visit to System Settings as a likely policy denial. The app cannot read a PPPC profile, so it names both causes and tells the user to ask IT. It never tries another capture path. Refs: REQ-APP-007, ADR-003, ADR-005
- Watch `kAudioHardwarePropertyProcessObjectList` and `kAudioProcessPropertyIsRunningInput` on each process object with Core Audio property listeners. The listener only records a timestamp and a process ID. A main actor task does the debounce. Refs: REQ-APP-008
- Map a bundle ID to a shown app name with a fixed table for known browser helpers, then with `NSRunningApplication` and the bundle name. A helper ID such as `com.google.Chrome.helper` maps to its parent browser by prefix. Refs: REQ-APP-008
- Debounce the prompt in a pure state machine. A mic session starts when any other process starts input. It ends when no other process runs input. The machine posts once per session, after about 3 seconds, and only when no recording runs and the setting is on. Refs: REQ-APP-008, REQ-APP-011
- Post notifications with `UserNotifications` and notification categories for the actions. If the user denies notifications, the popover shows the same actions. Refs: REQ-APP-008, REQ-APP-009
- Copy for Claude Code writes `/summarize-meeting "<path>"` to `NSPasteboard`. The app runs no LLM and no Claude Code process. Refs: REQ-APP-009, ADR-007
- Build the recent list from a directory listing of the output folder, sorted by modification date, with `.md.partial` files skipped. No database. Refs: REQ-APP-010, ADR-006
- Keep settings in `UserDefaults` under the app bundle ID. Defaults: output folder `~/Documents/Meetings`, mic-use prompt on, consent reminder off. Refs: REQ-APP-011
- Extend the no-network source scan to the app target. Refs: REQ-APP-012, ADR-007

## Risks / Trade-offs

- TCC attribution. The prompt may name the app, or it may name the process that started it. Mitigation: start the gate build from Finder and from `open`, and record which app each prompt names in `docs/findings.md`.
- Signing resets. An ad hoc build can lose its grants on each rebuild. A self-signed identity may also fail to keep grants on some macOS versions. Mitigation: the gate rebuilds with the stable identity and records whether the grants survive.
- MDM policies. A PPPC profile can deny the microphone or system audio with no prompt. The app cannot tell a policy denial from a user denial for system audio. Mitigation: measure signal, name both causes, and send the user to IT. No workaround.
- Notification permission. The user can deny notifications, or a profile can turn them off. Then the mic-use prompt and the saved notice do not show. Mitigation: the popover shows the saved path and its actions. The menu shows a note that the mic-use prompt needs notifications.
- Browser helper detection. A browser shows a helper process, not the call site. The app cannot tell a call from another web page that uses the microphone. The helper IDs can change between browser versions. Mitigation: name the browser only, keep the helper table small and tested, and fall back to the process name.
- Native app IDs are an assumption. Mitigation: the gate records the bundle IDs that real call apps show.
- Self-test sound. The Tink self-test plays a sound out loud. Mitigation: the step says so before it plays.
