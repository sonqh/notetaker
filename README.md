# notetaker

notetaker is a macOS command-line tool. It records a meeting, transcribes it on your Mac, and writes the transcript as one Markdown file.

It does not join the call as a bot. It sends no audio anywhere. It does not write a summary. You give the Markdown file to Claude Code for that.

Status: planned. There is no code yet. `docs/product.md` holds the roadmap. Each milestone is an OpenSpec change in `openspec/changes/`.

## The problem

- A notetaker bot joins your call, and everyone sees it.
- Hosted notetakers send your meeting audio to a vendor for transcription.
- Typing notes by hand means you stop listening.

## What it does

| You do | The tool does |
|---|---|
| Run `notetaker record` before the meeting | Captures system audio as "Them" and your microphone as "Me" |
| Talk | Transcribes both streams live, on device, with Apple's Speech framework |
| Type a line during the meeting | Saves it as a note with a timestamp |
| Type `/stop` or press Ctrl-C after the meeting | Writes one Markdown file with your notes and the transcript |
| Run `/summarize-meeting` in Claude Code | Claude writes key points, decisions, and action items |

## Requirements

- macOS 26 or later on Apple Silicon.
- Xcode 27 or a Swift 6.2+ toolchain, to build.
- English meetings only.
- Microphone permission.
- System audio recording permission.
- Headphones. With laptop speakers, your microphone hears the other side, and the same words show up under "Me" and "Them".

## Install

```bash
swift build -c release
mkdir -p ~/.local/bin
cp .build/release/notetaker ~/.local/bin/
```

Put `~/.local/bin` on your `PATH` if it is not there.

## Permissions

The first run asks for two permissions:

- Microphone, for "Me".
- System audio recording, for "Them".

macOS can attach these permissions to your terminal app (Terminal, iTerm, or VS Code) and not to `notetaker`. If you switch terminal apps, you may need to allow them again. `docs/findings.md` records what we observed.

A missing system audio permission gives silence with no error. Run `notetaker doctor` to check. It plays a short system sound and checks that the tool hears it.

## Usage

```bash
notetaker doctor
notetaker record --title "Standup"
```

`record` prints the live transcript. Type a line and press Enter to add a note. Type `/stop` or press Ctrl-C to stop. The tool then writes `~/Documents/Meetings/2026-11-03_1400_standup.md`.

| Option | Meaning | Default |
|---|---|---|
| `--title T` | Meeting title, also used in the file name | `Meeting` |
| `--out DIR` | Output folder | `~/Documents/Meetings` |
| `--keep-audio` | Also keep `<name>.me.caf` and `<name>.them.caf` | off |
| `--silence-timeout N` | Seconds without signal before a stream counts as dead | 5 |

`notetaker debug-audio` prints signal levels for both streams for 10 seconds. Use it when the transcript stays empty.

`docs/spec.md` lists the exit codes.

## Output

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
---

# Standup

## Notes

[00:03:10] Ask about the release date

## Transcript

[00:00:04] Me: Morning, can everyone hear me?
[00:00:09] Them: Yes, loud and clear.
```

- Timestamps count from the start of the recording.
- "Them" is all system audio. It can hold several people, and also notification sounds.
- If a stream fails during the meeting, the file gets a `warnings:` list.
- The tool writes the file as it goes, to a `.md.partial` file. It renames the file to `.md` on a clean stop. A crash leaves the partial file with every line so far.

## Summaries

The summary step is a Claude Code command. It lives in `.claude/commands/summarize-meeting.md`. Copy it to `~/.claude/commands/` to use it from any folder.

```text
/summarize-meeting ~/Documents/Meetings/2026-11-03_1400_standup.md
```

The command writes `2026-11-03_1400_standup.summary.md` next to the transcript. Each point carries the timestamp where someone said it. The command does not guess owners or due dates.

## Privacy

- The tool makes no network calls.
- The only network use is the OS downloading Apple's English speech model, if your Mac does not have it.
- A unit test fails if the source references a networking API.
- The tool keeps no audio unless you pass `--keep-audio`.

The summary step is separate. Claude Code sends the transcript to the service it uses, so follow your own rules for work meetings.

## Why "Me" and "Them"

The microphone is one stream and system audio is the other. That gives two speaker labels with no diarization model. The tool cannot tell apart two people on the far side of the call.

## Docs

- `docs/product.md`: direction, roadmap, cut line, non-goals.
- `docs/spec.md`: a summary of the behavior, the file format, the commands, and the exit codes.
- `openspec/changes/`: one change per milestone, with requirements, design, and tasks.
- `decisions/`: decision records.
- `docs/findings.md`: measurements and a manual test table. The build fills it in.
- `AGENTS.md`: rules and workflow for coding agents. The repo uses OpenSpec with spec-ctx.
