# Tasks

## 1. File name and collisions

- [ ] 1.1 Add the slug function and the file name builder. Test the default title, symbols, an empty slug, and the 40-character cut. [REQ-OUT-001] Verify: `swift test --filter FileNameTests`
- [ ] 1.2 Add the name picker. It treats `.md`, `.md.partial`, and the `.caf` names as used and adds `-2`, `-3`. Test it on a temp folder. [REQ-OUT-002] Verify: `swift test --filter NameCollisionTests`

## 2. Pure writer

- [ ] 2.1 Add the `TranscriptDocument` value and the front matter render. Cover `speakers`, `audio_route`, `duration`, `warnings`, and title quotes. Test with and without warnings. [REQ-OUT-003] Verify: `swift test --filter FrontMatterTests`
- [ ] 2.2 Add the pure map from the output device to `audio_route`. Test each device kind and an unknown device. [REQ-OUT-003] Verify: `swift test --filter AudioRouteTests`
- [ ] 2.3 Add the body render with the title heading, the optional `## Notes` section, and `## Transcript`. Test with notes and with no notes. [REQ-OUT-004] Verify: `swift test --filter NotesSectionTests`
- [ ] 2.4 Add the sort and the merge of segments into lines. Test order across streams, a late segment, the 2-second rule, and a Them segment between two Me segments. [REQ-OUT-005] Verify: `swift test --filter MergeOrderTests`
- [ ] 2.5 Add the `word[?]` marks. Take the threshold from the M2 entry in `docs/findings.md`. Test marks and segments with no confidence. [REQ-OUT-006] Verify: `swift test --filter ConfidenceMarkTests`

## 3. File store and writer actor

- [ ] 3.1 Add the file store. It writes `<name>.md.partial.tmp`, calls `fsync`, and renames the temp file over the target. Test UTF-8, `\n` line ends, and the last newline. [REQ-OUT-007] [REQ-OUT-008] Verify: `swift test --filter FileStoreTests`
- [ ] 3.2 Add the writer actor. It takes segments, notes, and warnings from one stream and rewrites the partial file. Test a late segment and read the file back. [REQ-OUT-007] Verify: `swift test --filter WriterActorTests`
- [ ] 3.3 Add the final write. It adds `duration`, renames to `<name>.md`, and deletes the partial file. Test a failed write that keeps the partial file. [REQ-OUT-008] [REQ-REC-006] Verify: `swift test --filter FinalWriteTests`

## 4. Record session

- [ ] 4.1 Add the preflight steps before t0 for the folder, the mic permission, and the model install from change 003. Put them behind protocols. Test exit codes 3, 7, and 8 on fakes. [REQ-REC-001] Verify: `swift test --filter RecordPreflightTests`
- [ ] 4.2 Add the session start. It joins the change 002 streams and the change 003 analyzers, sets t0, and writes the first partial file. Test a start failure on synthetic chunks. [REQ-REC-002] Verify: `swift test --filter RecordStartTests`
- [ ] 4.3 Add the stdin line reader for `/stop`, notes, empty lines, and end of input. Test it with a pipe. [REQ-REC-004] Verify: `swift test --filter StdinNotesTests`
- [ ] 4.4 Add the terminal printer for finalized lines, the volatile line, and stderr warnings. Test it on a fake terminal and on a pipe. [REQ-REC-003] Verify: `swift test --filter TerminalOutputTests`
- [ ] 4.5 Add the SIGINT source and the clean stop. Test the stop order and exit codes 0 and 9 on fakes. [REQ-REC-005] [REQ-REC-006] Verify: `swift test --filter CleanStopTests`
- [ ] 4.6 Map watchdog events from change 002 to warnings and recovery notes. Keep the other stream running. Test it on synthetic chunks. [REQ-REC-007] Verify: `swift test --filter StreamFailureTests`
- [ ] 4.7 Add the `--keep-audio` consumer for `.me.caf` and `.them.caf` as float32 at the native rate. Test it with synthetic chunks and read the files back. [REQ-REC-008] Verify: `swift test --filter KeepAudioTests`
- [ ] 4.8 Wire `notetaker record` to the session with `--title`, `--out`, `--keep-audio`, and `--silence-timeout`. [REQ-REC-002] [REQ-REC-006] Verify: `swift build && swift test --filter RecordCommandTests`

## 5. Docs

- [ ] 5.1 Update `docs/spec.md` and `README.md`. Add the new front matter fields, the two sections, the `[?]` marks, and stop by Ctrl-C or `/stop`. [REQ-OUT-003] [REQ-OUT-004] [REQ-REC-004] Verify: `specctx lint tells docs/spec.md README.md < /dev/null && specctx lint ste docs/spec.md README.md < /dev/null`
- [ ] 5.2 Update `.claude/commands/summarize-meeting.md`. Notes rank above the transcript, and the summary keeps the note headings. The command reads `speakers` and `audio_route`. [REQ-OUT-003] [REQ-OUT-004] Verify: `grep -q audio_route .claude/commands/summarize-meeting.md && grep -q '## Notes' .claude/commands/summarize-meeting.md`

## 6. Hardware gate

- [ ] 6.1 Write `scripts/gates/004-record.sh` for the M3 gate. It needs the dev Mac with audio. It runs 5 minutes of `say` and prints the file head and line count. It sends notes and `/stop` through a FIFO. It tests `kill -9` and Ctrl-C. [REQ-REC-002] [REQ-REC-004] [REQ-REC-005] [REQ-OUT-007] [REQ-OUT-008] Verify: `scripts/gates/004-record.sh`
