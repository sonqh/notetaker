# Specification

## Goal

One small tool. You start it before a meeting and stop it after. It captures the meeting audio with no bot, transcribes it live on your Mac, and writes one Markdown file.

The tool does not summarize. You give the file to Claude Code for that.

```text
notetaker record --title "Standup"
-> ~/Documents/Meetings/2026-11-03_1400_standup.md
```

## Decisions

The owner made these decisions. Do not reopen them without asking.

- macOS only. Target macOS 26 or later. The developer runs macOS 27 on Apple Silicon. If the toolchain cannot target macOS 26, use the lowest version it accepts and report it.
- Native Swift in a Swift package. No third-party dependency. No backend.
- English only.
- System audio is "Them", captured with a Core Audio process tap. The microphone is "Me", captured with AVAudioEngine. There is no diarization model.
- Transcription uses Apple's on-device speech framework: `SpeechAnalyzer` and `SpeechTranscriber`, with `SpeechDetector` to gate silence. Check the current SDK docs for exact API names, because the developer's OS is newer than most tutorials.
- The tool makes no network call. The OS may download Apple's speech model on first use. That is the only allowed network use.

## Why capture is the risky part

Open-source projects in this space report the same failure. macOS audio APIs return success and deliver silence. Causes include a denied permission, the wrong device, and a conflict with the echo canceller.

So the tool measures signal. It does not trust a return code.

## Output file

The file name is `<YYYY-MM-DD>_<HHmm>_<slug>.md`. The default folder is `~/Documents/Meetings`. The `--out` option changes it.

```markdown
---
title: Standup
date: 2026-11-03T14:00:00+07:00
duration: 00:32:10
language: en
source: notetaker 0.1, Apple SpeechAnalyzer (on-device)
---

# Standup

[00:00:04] Me: Morning, can everyone hear me?
[00:00:09] Them: Yes, loud and clear.
```

Rules:

- Timestamps are relative to the recording start, in `HH:MM:SS`.
- Lines are in order of start time across both streams.
- Merge consecutive finalized segments from the same speaker when the gap is under 2 seconds.
- Write only finalized recognition results to the file. Show volatile (interim) results in the terminal only.
- Write to `<name>.md.partial` after each finalized segment. Rename it to `.md` on a clean stop.
- If a stream fails during recording, finish the file and add a `warnings:` entry to the front matter.
- The file is plain UTF-8 Markdown that an LLM reads easily. It has no embedded audio and no JSON sidecar.

## Commands

### `notetaker doctor`

Reports:

- the macOS version
- the microphone permission
- whether a process tap can be created
- whether the Mac has the English speech model
- the default input and output devices

It exits non-zero if anything blocks recording. It names the missing permission or asset.

### `notetaker record`

```text
notetaker record [--title T] [--out DIR] [--keep-audio] [--silence-timeout N]
```

- Records, prints the live transcript, and writes the Markdown file.
- Stops on Enter or Ctrl-C.
- Keeps no audio unless you pass `--keep-audio`. Kept audio goes next to the file as `<name>.me.*` and `<name>.them.*`.
- `--silence-timeout N` sets how many seconds of silence mark a stream as dead. The default is 5.

### `notetaker debug-audio`

- Runs for 10 seconds.
- Prints peak, RMS, callback count, and non-zero sample count for each stream, once per second.
- Prints device and route-change events.
- Writes nothing.

## Suggested layout

- `NotetakerKit`: a library with capture, transcription, and the Markdown writer.
- `notetaker`: a thin executable.

Keep the logic testable without audio hardware.

## Non-goals

- Any LLM or summary feature.
- Network calls.
- A GUI or menu bar app.
- Provider abstractions.
- A database.
- Speaker identification beyond "Me" and "Them".
- Translation or any language other than English.
- Auto-update and analytics.
