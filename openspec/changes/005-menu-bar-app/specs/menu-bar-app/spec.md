# Spec Delta

## Purpose

The menu bar app is the friendly front end of notetaker. It starts and stops a recording, checks permissions by measured signal, and offers to take notes when a call starts. It installs and runs at user level on a managed Mac.

## ADDED Requirements

### Requirement: REQ-APP-001 Bundle the app with no Xcode project
The script `scripts/bundle.sh` MUST build `Notetaker.app` from the Swift package with no Xcode project. The bundle MUST carry an Info.plist with a bundle identifier, `LSUIElement` set to true, and both audio usage strings. The script MUST install the app to `~/Applications` and MUST NOT use `sudo` or write outside the user's home folder and the repo.

Refs: ADR-001, ADR-004, ADR-005

#### Scenario: SCN-APP-001-1 Build and install as a normal user
- **WHEN** a user with no admin rights runs `scripts/bundle.sh`
- **THEN** the script runs a release build, writes the binary to `Contents/MacOS`, and installs the bundle at `~/Applications/Notetaker.app`
- **AND** the script never calls `sudo` and never writes to `/Applications`

#### Scenario: SCN-APP-001-2 Info.plist content
- **WHEN** the script finishes
- **THEN** `Contents/Info.plist` holds `CFBundleIdentifier`, `LSUIElement` set to true, `NSMicrophoneUsageDescription`, and `NSAudioCaptureUsageDescription`

#### Scenario: SCN-APP-001-3 No background install
- **WHEN** the script finishes
- **THEN** it adds no login item, no LaunchAgent, and no driver

### Requirement: REQ-APP-002 Sign with a stable identity
The script MUST sign the app with a code signing identity from the user's login keychain, selected by name through an environment variable. With no identity, it MUST sign ad hoc and print a warning that permission grants can reset after each rebuild. The script MUST NOT create certificates or change TCC state.

Refs: REQ-APP-001, ADR-005

#### Scenario: SCN-APP-002-1 Named identity
- **WHEN** the environment variable names an identity that exists in the login keychain
- **THEN** the script signs the bundle with that identity and `codesign --verify` passes

#### Scenario: SCN-APP-002-2 Ad hoc fallback
- **WHEN** the environment variable is not set
- **THEN** the script signs the bundle ad hoc
- **AND** it prints a warning that macOS can ask for the permissions again after each rebuild

#### Scenario: SCN-APP-002-3 Missing identity
- **WHEN** the environment variable names an identity that the keychain does not have
- **THEN** the script exits with a non-zero code and names the missing identity
- **AND** it does not create a certificate

#### Scenario: SCN-APP-002-4 Grants survive a rebuild
- **WHEN** the user rebuilds the app with the same stable identity
- **THEN** macOS keeps the microphone and system audio grants, and the gate writes the result to `docs/findings.md`

### Requirement: REQ-APP-003 Menu bar icon states
The app MUST show a menu bar icon in one of three states: idle, recording, or warning. The state MUST follow the recording session and the watchdog states from NotetakerKit. The warning state MUST win over the recording state.

Refs: REQ-APP-012, ADR-003, ADR-004

#### Scenario: SCN-APP-003-1 Idle
- **WHEN** no recording runs
- **THEN** the icon shows the idle state

#### Scenario: SCN-APP-003-2 Recording
- **WHEN** a recording runs and no stream has a warning
- **THEN** the icon shows the recording state

#### Scenario: SCN-APP-003-3 Warning
- **WHEN** a recording runs and a stream is dead or suspect
- **THEN** the icon shows the warning state

#### Scenario: SCN-APP-003-4 Recovery
- **WHEN** the stream with the warning goes live again
- **THEN** the icon goes back to the recording state

### Requirement: REQ-APP-004 Recording popover
The popover MUST let the user start and stop a recording with an optional title. During a recording, it MUST show a Me meter, a Them meter, and the elapsed time. It MUST also show the last few finalized lines, the current volatile line, and the active warnings.

Refs: REQ-APP-003, REQ-APP-012, ADR-003

#### Scenario: SCN-APP-004-1 Start with a title
- **WHEN** the user types a title and presses Start
- **THEN** a recording starts and the file name uses the slug of that title

#### Scenario: SCN-APP-004-2 Start with no title
- **WHEN** the user presses Start with an empty title
- **THEN** the recording uses the default title `Meeting`

#### Scenario: SCN-APP-004-3 Live view
- **WHEN** a recording runs
- **THEN** the popover shows both meters, the elapsed time, the last finalized lines, the current volatile line, and each active warning

#### Scenario: SCN-APP-004-4 Stop
- **WHEN** the user presses Stop
- **THEN** the app finishes both analyzers, writes the final file, and removes the partial file

### Requirement: REQ-APP-005 Timestamped notes in the popover
The popover MUST have a notes field while a recording runs. Each note the user submits MUST get a timestamp relative to t0. The note MUST go to the file in the same form as a CLI note.

Refs: REQ-APP-004, REQ-APP-012, ADR-006

#### Scenario: SCN-APP-005-1 Add a note
- **WHEN** the user submits a note 2 minutes and 5 seconds after t0
- **THEN** the file holds that note with the timestamp `[00:02:05]`

#### Scenario: SCN-APP-005-2 Same form as the CLI
- **WHEN** the app and the CLI each record the same note at the same offset
- **THEN** the two note lines in the files are equal

#### Scenario: SCN-APP-005-3 No recording
- **WHEN** no recording runs
- **THEN** the notes field is not available

### Requirement: REQ-APP-006 First-run onboarding by measured signal
On first run, the app MUST show an onboarding window with three steps: microphone, system audio, and English speech model. A step MUST turn green only after the app measures signal or finds the model installed. A failed step MUST offer a button that opens the matching System Settings pane and does nothing else.

Refs: REQ-APP-007, ADR-002, ADR-003

#### Scenario: SCN-APP-006-1 Microphone step
- **WHEN** the user grants microphone access and says hello
- **THEN** the step shows a live meter and turns green only after the Me peak goes above the signal threshold

#### Scenario: SCN-APP-006-2 System audio step
- **WHEN** the app starts the tap and plays Tink in a separate process
- **THEN** the step turns green only if the Them peak goes above 0.01

#### Scenario: SCN-APP-006-3 Silence after a grant
- **WHEN** a permission call returns success but the measured signal stays at zero
- **THEN** the step stays not green and says "macOS sent silence. The permission is probably off."

#### Scenario: SCN-APP-006-4 Speech model step
- **WHEN** the `en_US` model is missing
- **THEN** the step shows the OS download progress and turns green only after the OS installs the model

### Requirement: REQ-APP-007 Managed Mac policy denial
When a management profile denies audio capture, the app MUST detect it by measured signal. It MUST tell the user that the Mac's management can block audio capture and that the user should ask IT. It MUST NOT try to work around the policy.

Refs: REQ-APP-006, ADR-003, ADR-005

#### Scenario: SCN-APP-007-1 Policy denies capture
- **WHEN** a step stays silent after the user tried the System Settings pane
- **THEN** the app says that a management profile can block audio capture and that the user should ask IT

#### Scenario: SCN-APP-007-2 No workaround
- **WHEN** a policy denies capture
- **THEN** the app does not change TCC state, does not install a profile, and does not switch to another capture method

### Requirement: REQ-APP-008 Mic-use prompt
The app MUST watch other processes for microphone input. When one app uses the microphone for about 3 seconds and no recording runs, the app MUST post one notification "<App> is using your mic. Start notes?" with Start and Dismiss. It MUST post at most one per mic session and MUST never start a recording on its own.

Refs: REQ-APP-004, REQ-APP-011

#### Scenario: SCN-APP-008-1 Native call app
- **WHEN** a native call app runs microphone input for 3 seconds
- **THEN** the app posts the notification with the name of the app

#### Scenario: SCN-APP-008-2 Browser helper
- **WHEN** a browser helper process such as `com.google.Chrome.helper` or `com.apple.WebKit.GPU` runs microphone input
- **THEN** the notification names the parent browser

#### Scenario: SCN-APP-008-3 Short use and repeat
- **WHEN** an app uses the microphone for less than 3 seconds, or the same mic session continues after a notification
- **THEN** the app posts no new notification

#### Scenario: SCN-APP-008-4 Start from the prompt
- **WHEN** the user picks Start in the notification
- **THEN** a recording starts, and picking Dismiss or doing nothing starts no recording

### Requirement: REQ-APP-009 Saved transcript notification
When a recording stops and the app writes the file, it MUST post a "Transcript saved" notification. The notification MUST offer three actions: Open, Show in Finder, and Copy for Claude Code.

Refs: REQ-APP-004, ADR-006, ADR-007

#### Scenario: SCN-APP-009-1 Open and Show in Finder
- **WHEN** the user picks Open or Show in Finder
- **THEN** the app opens the file in the default Markdown app or shows it in Finder

#### Scenario: SCN-APP-009-2 Copy for Claude Code
- **WHEN** the user picks Copy for Claude Code
- **THEN** the clipboard holds `/summarize-meeting "<path>"` with the absolute path of the file

#### Scenario: SCN-APP-009-3 Notifications denied
- **WHEN** the user denied notifications
- **THEN** the popover shows the saved path and the same three actions

### Requirement: REQ-APP-010 Recent transcripts menu
The menu MUST list the last 5 transcripts in the output folder, newest first. It MUST skip partial files. Picking an entry MUST open the file.

Refs: REQ-APP-011, ADR-006

#### Scenario: SCN-APP-010-1 Newest five
- **WHEN** the output folder holds 7 transcripts
- **THEN** the menu lists the 5 newest, newest first

#### Scenario: SCN-APP-010-2 Skip partial files
- **WHEN** the folder holds a `.md.partial` file
- **THEN** the menu does not list it

#### Scenario: SCN-APP-010-3 Empty folder
- **WHEN** the folder holds no transcript
- **THEN** the menu shows that no transcript exists yet

### Requirement: REQ-APP-011 Settings
The app MUST have settings for the output folder, the mic-use prompt, and the consent reminder. The mic-use prompt MUST be on by default. The consent reminder MUST be off by default. When it is on, the popover MUST show one quiet line during a recording: "Tell participants you are taking a transcript."

Refs: REQ-APP-008, REQ-APP-010

#### Scenario: SCN-APP-011-1 Output folder
- **WHEN** the user picks a new output folder
- **THEN** the next recording and the recent list use that folder

#### Scenario: SCN-APP-011-2 Prompt off
- **WHEN** the user turns the mic-use prompt off
- **THEN** the app posts no mic-use notification

#### Scenario: SCN-APP-011-3 Consent reminder
- **WHEN** the consent reminder is on and a recording runs
- **THEN** the popover shows the line "Tell participants you are taking a transcript."

#### Scenario: SCN-APP-011-4 Defaults
- **WHEN** the app runs for the first time
- **THEN** the mic-use prompt is on, the consent reminder is off, and the output folder is `~/Documents/Meetings`

### Requirement: REQ-APP-012 One core and one file format with the CLI
The app MUST use NotetakerKit for capture, the watchdog, transcription, and the file writer. It MUST NOT keep its own copy of that logic. A recording from the app MUST produce the same file format as a CLI recording. The app MUST make no network call.

Refs: ADR-004, ADR-006, ADR-007

#### Scenario: SCN-APP-012-1 Same format
- **WHEN** the app and the CLI each write a file from the same segments and notes
- **THEN** the two files are equal apart from the date and the duration

#### Scenario: SCN-APP-012-2 Recording from the prompt
- **WHEN** a recording starts from the mic-use prompt and stops
- **THEN** the file has the same name rule, front matter, and line format as a CLI file

#### Scenario: SCN-APP-012-3 No network
- **WHEN** the no-network test scans the app sources
- **THEN** it finds no networking API
