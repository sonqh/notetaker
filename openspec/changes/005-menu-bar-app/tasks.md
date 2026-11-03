# Tasks

## 1. Target, bundle, and signing

- [ ] 1.1 Add the `Notetaker` app target to `Package.swift` with a `MenuBarExtra` scene and an empty popover. It depends only on NotetakerKit. Extend the no-network source scan to the app sources. [REQ-APP-012] Verify: `swift build && swift test --filter NoNetworkTests`
- [ ] 1.2 Write `scripts/bundle.sh`. It runs `swift build -c release`, makes the bundle folders, writes `Info.plist` with `CFBundleIdentifier`, `LSUIElement`, and both usage strings, and installs to `~/Applications` with no `sudo`. Add `scripts/gates/005-bundle.sh` to test the layout, the plist keys, and the absence of `sudo`, a login item, and a LaunchAgent. [REQ-APP-001] Verify: `scripts/gates/005-bundle.sh`
- [ ] 1.3 Add signing to `scripts/bundle.sh`. Read the identity name from `NOTETAKER_SIGN_IDENTITY`. Sign ad hoc with a warning when it is not set. Exit non-zero and name the identity when the keychain does not have it. Add `scripts/gates/005-sign.sh` for the three cases. [REQ-APP-002] Verify: `scripts/gates/005-sign.sh`
- [ ] 1.4 Remove the "A GUI or menu bar app" non-goal from `docs/spec.md`. Add app install, Keychain Access identity steps, and a managed Mac section to `README.md`. [REQ-APP-001, REQ-APP-002, REQ-APP-007] Verify: `! grep -q "A GUI or menu bar app" docs/spec.md && grep -q "NOTETAKER_SIGN_IDENTITY" README.md && specctx lint ste README.md docs/spec.md`

## 2. View models with no hardware

- [ ] 2.1 Build the icon state machine from the session state and the two watchdog states. Warning wins over recording. Test idle, recording, warning, and recovery. [REQ-APP-003] Verify: `swift test --filter MenuBarStateTests`
- [ ] 2.2 Build the settings store over `UserDefaults` with the defaults: `~/Documents/Meetings`, mic-use prompt on, consent reminder off. [REQ-APP-011] Verify: `swift test --filter AppSettingsTests`
- [ ] 2.3 Build the recent transcripts list: the 5 newest `.md` files in the output folder, newest first, with partial files skipped and an empty state. [REQ-APP-010] Verify: `swift test --filter RecentTranscriptsTests`
- [ ] 2.4 Build the bundle ID to app name map. Map `com.google.Chrome.helper`, `com.apple.WebKit.GPU`, and the other known helpers to their browser. Fall back to the bundle name, then to the process name. [REQ-APP-008] Verify: `swift test --filter AppNameMapTests`
- [ ] 2.5 Build the mic-use prompt state machine with an injected clock. Post once per mic session after about 3 seconds. Post nothing when a recording runs or the setting is off. [REQ-APP-008, REQ-APP-011] Verify: `swift test --filter MicPromptTests`
- [ ] 2.6 Build the session view model over a fake NotetakerKit session. It covers start with an optional title, stop, and elapsed time. It also covers the last finalized lines, the volatile line, warnings, and the consent line. [REQ-APP-004, REQ-APP-011] Verify: `swift test --filter SessionViewModelTests`
- [ ] 2.7 Add the notes field model. Each note goes through the NotetakerKit notes API with its t0 offset. The field is not available with no recording. [REQ-APP-005] Verify: `swift test --filter AppNotesTests`
- [ ] 2.8 Add a format parity test. Feed the same segments and notes through the app session path and the CLI path, and compare the files apart from date and duration. [REQ-APP-012, REQ-APP-005] Verify: `swift test --filter FormatParityTests`
- [ ] 2.9 Build the saved actions model: Open, Show in Finder, and the Copy for Claude Code string `/summarize-meeting "<path>"` with an absolute path. [REQ-APP-009] Verify: `swift test --filter SavedActionsTests`
- [ ] 2.10 Build the onboarding step model. A step turns green only on measured signal or an installed model. Silence after a grant gives the silence message. Silence after a visit to System Settings gives the ask IT message. [REQ-APP-006, REQ-APP-007] Verify: `swift test --filter OnboardingTests`

## 3. UI and system wiring

- [ ] 3.1 Wire the menu bar icon, the popover, and the recent list to the view models. Throttle meter updates to about 15 Hz on the main actor. Needs the dev Mac. [REQ-APP-003, REQ-APP-004, REQ-APP-010] Verify: `scripts/gates/005-popover.sh`
- [ ] 3.2 Build the onboarding window with the microphone, system audio, and speech model steps. The System Settings button only opens the pane. Needs the dev Mac. [REQ-APP-006, REQ-APP-007] Verify: `scripts/gates/005-onboarding.sh`
- [ ] 3.3 Add the Core Audio watcher for `kAudioProcessPropertyIsRunningInput` on other processes and the mic-use notification with Start and Dismiss. Record the bundle IDs that real call apps show. Needs the dev Mac. [REQ-APP-008] Verify: `scripts/gates/005-mic-prompt.sh`
- [ ] 3.4 Post the "Transcript saved" notification with Open, Show in Finder, and Copy for Claude Code. Show the same actions in the popover when the user denies notifications. Needs the dev Mac. [REQ-APP-009] Verify: `scripts/gates/005-saved.sh`

## 4. Gate

- [ ] 4.1 Run the change gate on the dev Mac. Open the app from `~/Applications`. Pass onboarding. Record which app each permission prompt names. Start a recording from the mic-use prompt and compare its file with a CLI file. Rebuild with the stable identity and record whether the grants survive. Write the results to `docs/findings.md`. Needs the dev Mac. [REQ-APP-001, REQ-APP-002, REQ-APP-006, REQ-APP-008, REQ-APP-012] Verify: `scripts/gates/005-gate.sh`
