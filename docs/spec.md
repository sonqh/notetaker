# Specification

This file sums up how the app behaves. The requirements live in the OpenSpec changes under `openspec/changes/`, and in `openspec/specs/` after archive. If this file and a change disagree, the change wins. `docs/product.md` holds the roadmap.

## Goal

One small command-line tool. You start it before a meeting and stop it after. It captures the meeting audio with no bot, transcribes it live on your Mac, and writes one Markdown file.

The tool does not summarize. You give the file to Claude Code for that.

```text
notetaker record --title "Standup"
-> ~/Documents/Meetings/2026-11-03_1400_standup.md
```

## Fixed decisions

The owner made these decisions. Do not reopen them without asking.

- macOS only. The deployment target is macOS 26 (`.macOS(.v26)` in `Package.swift`). The dev machine runs macOS 27 on Apple Silicon with Xcode 27 and Swift 6.4.
- Native Swift in one Swift package. No third-party dependency. No backend.
- English only. The locale is `en_US`.
- Two streams, two speaker labels:
  - "Them" is system audio, captured with a Core Audio process tap.
  - "Me" is the microphone, captured with `AVAudioEngine`.
  - There is no diarization model.
- Transcription uses Apple's on-device Speech framework: `SpeechAnalyzer` with a `SpeechTranscriber` module. `SpeechDetector` gates silence on the Me stream.
- The tool makes no network call. The OS may download Apple's speech model. That is the only allowed network use, and the tool does not start it on its own (see `doctor`).

## Terms

| Term | Meaning |
|---|---|
| Stream | One audio source: Me or Them. |
| Chunk | A block of samples from one stream, with its sample rate, channel count, and the host time of its first sample. |
| Recording start (t0) | The host time when both captures of `record` run. All timestamps count from t0. |
| Segment | One finalized recognition result: speaker, start time, end time, text. |
| Line | One line in the output file. It holds one or more merged segments. |
| Volatile result | An interim recognition result. It can still change. |

## Why capture is the risky part

macOS audio APIs can return success and deliver silence. Known causes are a denied permission, the wrong device, and a conflict with the echo canceller.

So the tool measures signal. It never trusts a return code alone.

## Capture

### Them (system audio)

- Create a global stereo process tap that excludes this process (`CATapDescription(stereoGlobalTapButExcludeProcesses:)`).
- Put the tap in a private aggregate device. A tap alone delivers no buffers.
- Destroy the aggregate device and the tap on every exit path that the process can catch.
- Them holds all system audio, not only the meeting app. Notification sounds and music also land in Them.

### Me (microphone)

- Use the `AVAudioEngine` input node on the default input device.
- Do not enable voice processing. It can stop the process tap.

### Shared rules

- Both streams use host time, so they stay aligned.
- The audio callback never blocks. It does not lock, allocate, write files, log, or `await`. It copies samples into a preallocated bounded ring buffer and counts overflow.
- A consumer task reads the ring buffer and yields chunks on an `AsyncStream`.
- Each consumer keeps its stream's timeline continuous. If samples drop on overflow, it inserts zeros of the same length and counts the drop. The transcript timestamps then stay right.

## Signal watchdog

The watchdog runs per stream. N is `--silence-timeout`, default 5 seconds.

Digital silence is normal on Them. When nobody talks and no app plays sound, the tap delivers exact zeros. A denied permission also delivers exact zeros. So the watchdog cannot treat zeros on Them as a failure by themselves.

| Stream | Condition for N seconds | State | Action in `record` |
|---|---|---|---|
| Me or Them | No callbacks | dead | At start: exit with the stream's code. Later: warning. |
| Me | Every sample is zero | dead | At start: exit 5. Later: warning. |
| Them | Every sample is zero, and no other process runs audio output | idle | Nothing. This is normal. |
| Them | Every sample is zero, and another process runs audio output | suspect | Print a warning that names the likely missing system audio permission. Do not exit. |
| Any | Signal returns | live | Print a recovery note if the stream had a warning. |

"Another process runs audio output" means `kAudioProcessPropertyIsRunningOutput` is true for a process other than this one. Some apps keep output running while they play silence. M1 measures how often this gives a false "suspect" state.

"At start" means the first N seconds after t0.

The `doctor` command gives the definite Them check. It plays a short system sound in a separate process and checks that the tap hears it.

## Transcription

- Run one `SpeechAnalyzer` per stream.
- Use a `SpeechTranscriber` with `en_US`, volatile results, and audio time ranges.
- Add a `SpeechDetector` to the Me analyzer. Room noise must not become invented sentences. M2 decides if Them also needs it.
- Convert chunks off the callback thread with `AVAudioConverter` to the format that `SpeechAnalyzer.bestAvailableAudioFormat` returns.
- Map each result's audio time range to t0: segment start = (host time of the stream's first chunk − t0) + result start.
- Check the current SDK for exact API names. The dev OS is newer than most tutorials. The SDK interface is the source of truth.

## Output file

### Name and folder

- The name is `<YYYY-MM-DD>_<HHmm>_<slug>.md`. Date and time are local time at t0.
- The slug is the title in lower case. Each run of characters other than `a-z` and `0-9` becomes one `-`. Strip `-` from both ends. Cut it to 40 characters. If it is empty, use `meeting`.
- The default title is `Meeting`.
- The default folder is `~/Documents/Meetings`. The `--out DIR` option changes it. The tool creates the folder if it is missing.
- If the file already exists, add `-2`, `-3`, and so on before `.md`. Never overwrite a file.

### Format

```markdown
---
title: Standup
date: 2026-11-03T14:00:00+07:00
duration: 00:32:10
language: en-US
source: notetaker 0.1, Apple SpeechAnalyzer (on-device)
speakers:
  Me: the user
  Them: all other audio, may be several people
audio_route: headphones
warnings:
  - "00:12:40 Them: no callbacks for 5 s"
---

# Standup

## Notes

[00:03:10] Ask about the release date

## Transcript

[00:00:04] Me: Morning, can everyone hear me?
[00:00:09] Them: Yes, loud and clear.
```

Rules:

- `date` is t0 in ISO 8601 with the local offset.
- `duration` is the time from t0 to stop, in `HH:MM:SS`.
- `warnings` is present only when there is at least one warning. Each entry starts with its timestamp.
- `audio_route` is `headphones` or `speakers`, from the output device at t0.
- `## Notes` holds the notes that the user typed, each with the time the user typed it. The section is present only when there are notes.
- A word with low recognition confidence ends in `[?]`. M2 measures the threshold.
- Each line starts with `[HH:MM:SS]`, the start time of its first segment, relative to t0.
- Lines are in order of start time across both streams.
- Merge two segments into one line when all three conditions are true:
  - They are next to each other in that order.
  - They have the same speaker.
  - The gap (next start − previous end) is under 2 seconds.
- Write only finalized results to the file. Show volatile results in the terminal only.
- The file is plain UTF-8 Markdown with `\n` line endings. It has no embedded audio and no JSON sidecar.

### Writing and crash safety

- After each finalized segment, rewrite the whole `<name>.md.partial` file. Write to a temporary file in the same folder, then rename it over the partial file. A crash, including `kill -9`, then always leaves a complete, readable partial file.
- A late segment from one stream can start before a segment that the other stream already wrote. The rewrite sorts all segments, so the late segment lands in the right place.
- The partial file has the same format. It has no `duration`.
- On a clean stop, finish both analyzers, write the final file with `duration`, and remove the partial file.
- If a stream fails during recording, keep recording the other stream. Add a `warnings` entry and print it in the terminal.

## Permissions

| Permission | Info.plist key | Needed by |
|---|---|---|
| Microphone | `NSMicrophoneUsageDescription` | Me |
| System audio recording | `NSAudioCaptureUsageDescription` | Them |

- A SwiftPM executable has no Info.plist. Embed one in the binary with a linker `__TEXT,__info_plist` section.
- macOS can attach the permission to the app that started the tool (Terminal, iTerm, or VS Code) and not to the binary. Then the permission changes with the terminal app you use. M1 measures this, and `README.md` documents it.
- No public API reports the system audio permission state. Only signal can prove it.
- The tool can trigger the standard permission prompts. It never changes System Settings or TCC state.

## Commands

### `notetaker doctor`

Reports one line per check, then exits.

| Check | How |
|---|---|
| macOS version | Fail if lower than 26. |
| Microphone permission | Report the state. Do not prompt. Fail if denied. |
| Process tap | Create and destroy a tap and an aggregate device. Fail if creation fails. |
| Them self-test | Start the tap, run `/usr/bin/afplay /System/Library/Sounds/Tink.aiff`, and check that the Them peak goes above 0.01. Fail if it does not, and name the system audio permission. |
| English speech model | Report whether the Mac has `en_US`, from `SpeechTranscriber.installedLocales`. Do not trust `AssetInventory.status` alone: it can say `supported` for an installed model. Fail if it is missing. Do not start a download. Say that `notetaker record` asks the OS to install it on first use. |
| Devices | Print the default input and output devices. |

It exits 0 when all checks pass. Otherwise it exits with the code of the first failed check.

### `notetaker record`

```text
notetaker record [--title T] [--out DIR] [--keep-audio] [--silence-timeout N]
```

- Records, prints the live transcript, and writes the Markdown file.
- Stops on the line `/stop` or on SIGINT (Ctrl-C). A second SIGINT during shutdown exits at once and keeps the partial file.
- Any other line that is not empty becomes a note, stamped with the time the user typed it. Empty lines do nothing.
- Requests the microphone permission if its state is "not determined".
- If the `en_US` model is missing, asks the OS to install it through `AssetInventory`, and prints progress. This is the only path that can cause a network download, and the OS performs it.
- Keeps no audio unless you pass `--keep-audio`. Then it writes `<name>.me.caf` and `<name>.them.caf` next to the Markdown file: 32-bit float PCM at the native sample rate of each stream. A consumer task writes them, never the audio callback.
- `--silence-timeout N` sets N for the watchdog. N is a whole number of seconds from 1 to 60. The default is 5.
- Terminal output: finalized lines in normal text, the current volatile result on one updating line, and warnings on stderr.

### `notetaker debug-audio`

```text
notetaker debug-audio [--seconds S]
```

- Runs both captures for S seconds. The default is 10.
- Prints, once per second for each stream: peak, RMS, callback count, non-zero sample count, overflow count, and watchdog state.
- Prints device and route change events.
- Writes no file.
- Exits 0 if both captures started, even if the signal is zero. The numbers are the result.

## Exit codes

| Code | Meaning |
|---|---|
| 0 | Clean stop with no warnings |
| 1 | Usage error, for example an unknown option |
| 2 | macOS version lower than 26 |
| 3 | Microphone permission denied |
| 4 | Process tap or aggregate device cannot be created |
| 5 | Me stream has no signal at start |
| 6 | Them stream has no callbacks at start, or the Them self-test failed |
| 7 | English speech model missing or cannot be installed |
| 8 | The app cannot write to the output folder, or a file write failed |
| 9 | Clean stop, but the file has warnings |
| 70 | Internal error |

## Layout

- `NotetakerKit`: a library with capture, the watchdog, transcription, and the Markdown writer.
- `notetaker`: a thin executable that parses arguments and calls `NotetakerKit`.
- `tap-probe` and `speech-probe`: dev tools from change 001. They do not install.
- The menu bar app: a thin front end from change 005.
- `NotetakerKitTests`: unit tests.

Tests must cover all logic that does not touch hardware, with no audio hardware. That logic is the level meter, the watchdog, segment merging and ordering, slug and file names, and front matter. Put hardware access behind small protocols so tests can feed synthetic chunks.

## Non-goals

- Any LLM or summary feature.
- Network calls.
- Provider abstractions.
- A database.
- Speaker identification beyond "Me" and "Them".
- A tap on one app only. Them is all system audio.
- Translation or any language other than English.
- Auto-update and analytics.
