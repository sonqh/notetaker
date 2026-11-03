# Spec Delta

## Purpose

Defines the Markdown file that one recording writes: its name, front matter, notes, transcript lines, the crash-safe partial file, and the final file.

## ADDED Requirements

### Requirement: REQ-OUT-001 File name and folder
The tool SHALL name the file `<YYYY-MM-DD>_<HHmm>_<slug>.md` from the local time at t0 and the title. The default title is `Meeting`. The slug is the lower-case title with each run of characters outside `a-z` and `0-9` changed to one `-`, with `-` stripped from both ends, cut to 40 characters. An empty slug becomes `meeting`. The default folder is `~/Documents/Meetings`. `--out DIR` changes it. The tool SHALL create a missing folder.

Refs: ADR-006

#### Scenario: SCN-OUT-001-1 Default title and folder
- **WHEN** `record` starts with no `--title` and no `--out` at 14:05 local time on 2026-11-03
- **THEN** the file path is `~/Documents/Meetings/2026-11-03_1405_meeting.md`

#### Scenario: SCN-OUT-001-2 Slug from a title with symbols
- **WHEN** the title is `  Q3 Planning: Roadmap & Budget!! `
- **THEN** the slug is `q3-planning-roadmap-budget`

#### Scenario: SCN-OUT-001-3 Title with no letters or digits
- **WHEN** the title is `!!!` or a title of only non-ASCII letters such as `会议`
- **THEN** the slug is `meeting`

#### Scenario: SCN-OUT-001-4 Long title
- **WHEN** the slug before the cut is longer than 40 characters
- **THEN** the slug keeps the first 40 characters
- **AND** the slug has no `-` at its end after the cut

#### Scenario: SCN-OUT-001-5 Missing output folder
- **WHEN** `--out DIR` names a folder that does not exist and its parent is writable
- **THEN** the tool creates `DIR` and writes the file in it

### Requirement: REQ-OUT-002 Never overwrite a file
The tool SHALL never overwrite an existing file. A name is in use when `<name>.md` or `<name>.md.partial` exists in the folder. For a name in use the tool SHALL add `-2`, `-3`, and so on before `.md` and use the first free name. Every file of one recording uses the same base name.

Refs: REQ-OUT-001, ADR-006

#### Scenario: SCN-OUT-002-1 One collision
- **WHEN** `2026-11-03_1405_standup.md` exists and a new recording gets the same name
- **THEN** the new file is `2026-11-03_1405_standup-2.md`

#### Scenario: SCN-OUT-002-2 Several collisions
- **WHEN** `2026-11-03_1405_standup.md` and `2026-11-03_1405_standup-2.md` both exist
- **THEN** the new file is `2026-11-03_1405_standup-3.md`

#### Scenario: SCN-OUT-002-3 Partial file from a crashed run
- **WHEN** only `2026-11-03_1405_standup.md.partial` exists from an earlier crashed run
- **THEN** the new recording uses `2026-11-03_1405_standup-2.md`
- **AND** the old partial file stays unchanged

### Requirement: REQ-OUT-003 Front matter
The file SHALL start with YAML front matter that holds `title`, `date`, `duration`, `language`, `source`, `speakers`, `audio_route`, and `warnings`. `date` is t0 in ISO 8601 with the local offset. `duration` is t0 to stop in `HH:MM:SS`. `speakers` names Me as the user and Them as all other audio. `audio_route` is `headphones` or `speakers` from the output device at t0. `warnings` is present only when a warning exists.

Refs: REQ-OUT-007, ADR-006

#### Scenario: SCN-OUT-003-1 Front matter with warnings
- **GIVEN** a final file with title `Standup`, t0 at 2026-11-03 14:00:00 in UTC+7, and a headphone route
- **WHEN** the stop comes 32 min 10 s later and Them has one warning at 00:12:40
- **THEN** the front matter is, line by line: `---`, `title: Standup`, `date: 2026-11-03T14:00:00+07:00`, `duration: 00:32:10`, `language: en-US`, `source: notetaker 0.1, Apple SpeechAnalyzer (on-device)`, `speakers:`, `  Me: the user`, `  Them: all other audio, may be several people`, `audio_route: headphones`, `warnings:`, `  - "00:12:40 Them: no callbacks for 5 s"`, `---`

#### Scenario: SCN-OUT-003-2 Front matter with no warnings
- **WHEN** the recording has no warning
- **THEN** the front matter has no `warnings` key

#### Scenario: SCN-OUT-003-3 Speaker route
- **WHEN** the default output device at t0 is the built-in speaker
- **THEN** `audio_route` is `speakers`
- **AND** a later change of the output device does not change the value

#### Scenario: SCN-OUT-003-4 Title that needs quotes
- **WHEN** the title is `Q3: plan "v2"`
- **THEN** the front matter holds `title: "Q3: plan \"v2\""`
- **AND** a YAML parser reads the title back as `Q3: plan "v2"`

### Requirement: REQ-OUT-004 Notes section
After the front matter and the `# <title>` heading, the file SHALL hold a `## Notes` section when the user typed at least one note. Each note is one line `[HH:MM:SS] <note text>`, where the time is relative to t0. Notes are in time order. The file SHALL hold no `## Notes` section when there are no notes.

Refs: REQ-REC-004, ADR-006

#### Scenario: SCN-OUT-004-1 Two notes
- **WHEN** the user typed `ask about budget` at 00:03:12 and `owner is Dana` at 00:07:45
- **THEN** the body is `# <title>`, a blank line, `## Notes`, a blank line, `[00:03:12] ask about budget`, `[00:07:45] owner is Dana`, a blank line, `## Transcript`, a blank line, and then the transcript lines

#### Scenario: SCN-OUT-004-2 No notes
- **WHEN** the user typed no note
- **THEN** the body is `# <title>`, a blank line, `## Transcript`, a blank line, and then the transcript lines

#### Scenario: SCN-OUT-004-3 Note text kept as typed
- **WHEN** a note holds Markdown such as `**todo** #42`
- **THEN** the file holds the text unchanged after the timestamp

### Requirement: REQ-OUT-005 Transcript order and merge
Under `## Transcript` each line SHALL be `[HH:MM:SS] <Speaker>: <text>`. The time is the start of its first segment relative to t0. Lines SHALL be in start-time order across both streams. Me comes before Them on equal start times. Two segments next to each other in that order SHALL merge into one line when they have the same speaker. The gap from the first end to the next start must be under 2 seconds.

Refs: REQ-OUT-007, ADR-006

#### Scenario: SCN-OUT-005-1 Order across streams
- **WHEN** the segments are Them at 9.0 s and Me at 4.0 s
- **THEN** the line `[00:00:04] Me: ...` comes before the line `[00:00:09] Them: ...`

#### Scenario: SCN-OUT-005-2 Late segment
- **WHEN** the file holds a Them segment at 10 s and later a Me segment at 6 s finalizes
- **THEN** the next rewrite puts the Me line before the Them line

#### Scenario: SCN-OUT-005-3 Merge under 2 seconds
- **WHEN** Me has a segment from 4.0 s to 6.0 s with text `Morning,` and a segment from 7.5 s to 9.0 s with text `can you hear me?`
- **THEN** the file holds one line `[00:00:04] Me: Morning, can you hear me?`

#### Scenario: SCN-OUT-005-4 No merge at 2 seconds or more
- **WHEN** the gap between two Me segments is 2.0 s
- **THEN** the file holds two Me lines

#### Scenario: SCN-OUT-005-5 No merge across another speaker
- **WHEN** a Them segment falls between two Me segments in start-time order
- **THEN** the file holds three lines and the two Me segments do not merge

### Requirement: REQ-OUT-006 Low-confidence marks
The tool SHALL write `word[?]` for each word whose confidence is below the threshold that M2 measured and recorded in `docs/findings.md`. Words at or above the threshold stay unmarked. When a segment has no confidence values, the tool SHALL write its words with no marks.

Refs: REQ-OUT-005, ADR-006

#### Scenario: SCN-OUT-006-1 One low-confidence word
- **WHEN** the threshold is 0.5 and a Them segment `ship it Friday` has confidence 0.9, 0.8, and 0.3
- **THEN** the line text is `ship it Friday[?]`

#### Scenario: SCN-OUT-006-2 No confidence values
- **WHEN** a segment has no confidence values
- **THEN** the line text has no `[?]` mark

### Requirement: REQ-OUT-007 Crash-safe partial file
During a recording the tool SHALL keep `<name>.md.partial` in the output folder. After each finalized segment, note, or warning, it SHALL write the whole file to a temp file in the same folder. Then it SHALL rename the temp file over the partial file. The partial file SHALL use the final format with no `duration`. Only finalized segments go in the file.

Refs: REQ-OUT-003, ADR-006

#### Scenario: SCN-OUT-007-1 Rewrite after a segment
- **WHEN** a segment finalizes
- **THEN** the partial file holds that segment in its sorted place, together with all earlier segments and notes

#### Scenario: SCN-OUT-007-2 Hard kill
- **WHEN** the process gets `kill -9` during a recording
- **THEN** `<name>.md.partial` exists and is a complete file in the format of this spec
- **AND** it holds every line that finalized before the last rewrite

#### Scenario: SCN-OUT-007-3 No duration and no volatile text
- **WHEN** a reader opens the partial file during a recording
- **THEN** its front matter has no `duration` key
- **AND** it holds no volatile result text

### Requirement: REQ-OUT-008 Final file
On a clean stop the tool SHALL write `<name>.md` with `duration` through a temp file and a rename, then remove `<name>.md.partial`. The file SHALL be UTF-8 with `\n` line ends and SHALL end with one `\n`.

Refs: REQ-OUT-007, REQ-REC-006, ADR-006

#### Scenario: SCN-OUT-008-1 Clean stop
- **WHEN** the recording stops cleanly
- **THEN** `<name>.md` exists with a `duration` key
- **AND** `<name>.md.partial` and the temp file do not exist

#### Scenario: SCN-OUT-008-2 Encoding
- **WHEN** the transcript holds the text `café – naïve`
- **THEN** the file bytes are valid UTF-8, hold no `\r`, and end with exactly one `\n`
