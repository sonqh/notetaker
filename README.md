# notetaker

notetaker is a macOS command-line tool. It records a meeting, transcribes it on your Mac, and writes the transcript as one Markdown file.

It does not join the call as a bot. It sends no audio anywhere. It does not write a summary. You give the Markdown file to Claude Code for that.

Status: planned. There is no code yet. `docs/spec.md` says what to build, and `docs/build-plan.md` says how to build it in three steps.

## The problem

- A notetaker bot joins your call, and everyone sees it.
- Hosted notetakers send your meeting audio to a vendor for transcription.
- Typing notes by hand means you stop listening.

## What it does

| You do | The tool does |
|---|---|
| Run `notetaker record` before the meeting | Captures system audio as "Them" and your microphone as "Me" |
| Talk | Transcribes both streams live, on device, with Apple's speech framework |
| Press Enter after the meeting | Writes one Markdown file with timestamps |
| Run `/summarize-meeting` in Claude Code | Claude writes key points, decisions, and action items |

## Try it

```bash
notetaker doctor
notetaker record --title "Standup"
```

The second command prints the live transcript in the terminal. It writes `~/Documents/Meetings/2026-11-03_1400_standup.md` when you press Enter or Ctrl-C.

## Output

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

The tool writes the file as it goes, to a `.md.partial` file. It renames the file to `.md` on a clean stop. A crash does not lose the transcript.

## Summaries

The summary step is a Claude Code command. It lives in `.claude/commands/summarize-meeting.md`. Copy it to `~/.claude/commands/` to use it from any folder.

```text
/summarize-meeting ~/Documents/Meetings/2026-11-03_1400_standup.md
```

The command writes `2026-11-03_1400_standup.summary.md` next to the transcript. Each point carries the timestamp where someone said it. The command does not guess owners or due dates.

## Requirements

- macOS 26 or later on Apple Silicon.
- English meetings only.
- Microphone permission.
- System audio recording permission.
- Headphones. With laptop speakers, your microphone hears the other side, and the same words show up under "Me" and "Them".

## Privacy

- The tool makes no network calls.
- The only network use is the OS downloading Apple's speech model the first time you transcribe.
- A test fails if the package references a networking API.
- The tool keeps no audio unless you pass `--keep-audio`.

Your summary step is separate. Claude Code sends the transcript to whatever service it uses, so follow your own rules for work meetings.

## Why "Me" and "Them"

The microphone is one stream and system audio is the other. That gives two speaker labels with no diarization model. The tool cannot tell apart two people on the far side of the call.

## Docs

- `docs/spec.md`: decisions, file format, commands, non-goals.
- `docs/build-plan.md`: milestones, known traps, acceptance criteria.
- `docs/findings.md`: exit codes, measurements, and a manual test table. The build fills it in.
- `AGENTS.md`: rules for coding agents.
