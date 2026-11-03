# Findings

The build fills in this file. Leave a section as "not yet measured" until someone measures it.

## Exit codes

`docs/spec.md` defines the exit codes. Record here any code that behaves differently from the spec, and why.

None yet.

## Audio format

Not yet measured. Record:

- the tap format (`kAudioTapPropertyFormat`): sample rate and channel count
- the microphone format
- the format that `SpeechAnalyzer.bestAvailableAudioFormat` returns. A scratch probe on 2026-11-03 got 1 channel, 16000 Hz, Int16, with and without `SpeechDetector`.
- the conversion the tool does

## Info.plist approach

Not yet measured. Record what worked, and which app the permission prompt names.

## Permission attribution

| Launched from | Microphone prompt names | System audio prompt names |
|---|---|---|
| Terminal | not tested | not tested |
| VS Code terminal | not tested | not tested |
| iTerm | not tested | not tested |

## Measurements

| Question | Result |
|---|---|
| Share of the known paragraph found in the Them lines | not yet measured |
| Me segments in 60 s of room silence | not yet measured |
| How often Me lines duplicate Them lines on laptop speakers | not yet measured |
| Chunk delivery latency, p50 and p95 | not yet measured |
| Time from speech to finalized line, p50 and p95 | not yet measured |
| False "suspect" states on Them in a quiet meeting | not yet measured |

## Gotchas

List every gotcha you hit, with the fix.

- `AssetInventory.status(forModules:)` returned `.supported` for a `SpeechTranscriber` with `en_US`, after a file transcribed. `SpeechTranscriber.installedLocales` listed `en_US`. Fix: the model check reads `installedLocales`. (Scratch probe on the dev Mac, 2026-11-03, during the planning of change 003.)
- `SpeechTranscriber` output has capitals and punctuation by default. It writes "three" as "3". Fix: recall fixtures have no numbers. (Same probe.)

## Manual test table

Mark a row "not tested" unless you ran it.

| Case | Result | Notes |
|---|---|---|
| Laptop speakers | not tested | |
| AirPods | not tested | |
| Wired headphones | not tested | |
| Zoom | not tested | |
| Google Meet | not tested | |
| Microsoft Teams | not tested | |
| AirPods connect mid-recording | not tested | |
| Unplug the microphone mid-recording | not tested | |
| `kill -9` mid-recording | not tested | |
