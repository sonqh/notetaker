# Proposal

## Why

Change 002 gives two chunk streams on one host-time timeline. The tool still prints no words. Milestone M2 turns those chunks into a live transcript on the Mac, with no network call. It also proves that the transcript is good enough to keep building. The M2 gate measures word recall, false Me segments in a silent room, and latency.

The SDK on the dev Mac differs from most tutorials. Some APIs are new in macOS 27 and the target is macOS 26. A probe on the dev Mac also showed that `AssetInventory.status(forModules:)` reports `supported` for `en_US` while `SpeechTranscriber.installedLocales` lists `en_US` and a file transcribes. So the model check needs care, and the plan must test its own assumptions.

## What Changes

- Run one `SpeechAnalyzer` per stream. Feed it the chunks from change 002.
- Convert each chunk with `AVAudioConverter` on a consumer task to the format that `SpeechAnalyzer.bestAvailableAudioFormat` returns. The audio callback does no work for transcription.
- Use a `SpeechTranscriber` for `en_US` with the `timeIndexedProgressiveTranscription` preset plus the `transcriptionConfidence` attribute.
- Pass `bufferStartTime` relative to t0, so every result lands on the t0 timeline.
- Gate the Me stream with a `SpeechDetector`. Decide by measurement if Them needs one too.
- Add a `Segment` value: speaker, start, end, text, and word confidences. Add a stream of volatile results for the terminal.
- Print finalized lines and one updating volatile line in the terminal. Do not write the Markdown file.
- Add the `en_US` model check to `doctor`. It never starts a download.
- Let `record` ask `AssetInventory` to install a missing model and print progress.
- Turn analyzer errors such as `insufficientResources` and `tooManyAssetLocalesAllocated` into stream warnings. The other stream keeps going.
- Put the analyzer behind a small protocol, so ordering and time mapping test with fake results.
- Compare `SpeechTranscriber` with `DictationTranscriber` (`.punctuation`, `.farField`) on the Me stream. Record the result in `docs/findings.md` and in ADR-008.
- Add gate scripts `scripts/gates/003-*.sh` for the checks that need the dev Mac with audio.

Out of scope:

- The Markdown file, the partial file, and the merge rule (change 004).
- Any GUI or menu bar app (change 005).
- Diarization or any speaker label beyond Me and Them.
- Any language other than English.
- Duplicate removal for laptop speakers. This change only measures it.
- Custom vocabulary. This change measures `contextualStrings` and adds no option for it.
- Any summary or LLM feature.

## Capabilities

### New Capabilities

- `transcription`: on-device live transcription of the Me and Them chunk streams. It also covers the segment model, the volatile result stream, the speech model check and install, analyzer errors, and the measured quality gate.

### Modified Capabilities

None. The `doctor` model check is a new requirement in `transcription`. It plugs into the check list of change 002.

## Impact

- New code in `Sources/NotetakerKit/Transcription/` and new tests in `Tests/NotetakerKitTests/`.
- `record` in `Sources/notetaker/` prints the live transcript. `doctor` gains one check row.
- New scripts `scripts/gates/003-recall.sh`, `scripts/gates/003-silence.sh`, `scripts/gates/003-sentence.sh`, and `scripts/gates/003-latency.sh`.
- A known paragraph of at least 150 words in `Tests/Fixtures/`.
- Updates to `docs/findings.md` and `decisions/ADR-008`.
- Frameworks: Speech, AVFoundation, CoreMedia from the SDK. No third-party package.
- Network: none from the tool. The OS may download the speech model, and only `record` asks it to.
