# Design

## Context

Change 002 yields one `AsyncStream` of chunks per stream. Each chunk has samples, sample rate, channel count, and the host time of its first sample. t0 is the host time when both captures run. Change 002 also fills dropped samples with zeros, so each stream's timeline stays continuous. Change 001 gives `speech-probe`, which transcribes a file and scores word recall and WER.

The target is macOS 26 and the dev Mac runs macOS 27. The SDK interface for Speech is the source of truth. A scratch probe on the dev Mac on 2026-11-03 showed these facts:

- `SpeechAnalyzer` is an actor. `AnalyzerInput(buffer:bufferStartTime:)` exists on 26.
- `timeIndexedProgressiveTranscription` holds reporting options `volatileResults` and `fastResults`, attribute `audioTimeRange`, and no transcription option. `transcriptionConfidence` is a separate attribute option on both transcribers.
- `bestAvailableAudioFormat` returns 1 channel, 16 kHz, Int16 for the transcriber alone and for the transcriber plus `SpeechDetector`.
- `SpeechTranscriber.installedLocales` lists `en_US`. `AssetInventory.status(forModules:)` returns `supported`, not `installed`, for the same transcriber. A file still transcribes. The status call returns `installed` for `SpeechDetector` alone.
- The default `SpeechTranscriber` output has capitals and punctuation. It wrote "three" as "3".
- Word confidence comes per text run, with values from about 0.4 to 1.0.
- `isFinal` is an extension on `SpeechModuleResult`. `DictationTranscriber` has `.punctuation`, `.farField`, and `frequentFinalization`, and no `fastResults`.
- `AnalyzerInputConverter`, `ignoresResourceLimits`, and `CaptureInputSequenceProvider` need macOS 27.

These assumptions are still open: whether `SpeechDetector` keeps non-speech out of the transcript, whether two analyzers run at once without a resource error, and whether `contextualStrings` helps.

## Goals / Non-Goals

**Goals:**

- Live Me and Them text in the terminal, with times on the t0 timeline.
- A segment model that change 004 can merge and write with no change.
- A model check that never downloads, and an install path only in `record`.
- Logic that tests with a fake engine on any Mac.
- Real numbers for recall, silent-room segments, and latency.

**Non-Goals:**

- The Markdown file, the merge rule, and the partial file.
- Duplicate removal for laptop speakers.
- A vocabulary option, diarization, or a language other than English.
- Any GUI.

## Decisions

- Wrap the framework in a `TranscriptionEngine` protocol with `prepare(format:)`, `feed(_:startTime:)`, `finish()`, and a results sequence of plain values (text, range, final flag, word confidences). The live type `SpeechAnalyzerEngine` holds the `SpeechAnalyzer` and its modules. Tests use `FakeEngine`. A direct use of the framework in the pipeline cannot run in `swift test` without the model. Refs: REQ-TRN-011, ADR-004
- Use one `SpeechAnalyzer` per stream with its own transcriber. One shared analyzer cannot keep the two speakers apart. Refs: REQ-TRN-001, ADR-008
- Build the transcriber with `SpeechTranscriber(locale:transcriptionOptions:reportingOptions:attributeOptions:)`, taking the preset's options and adding `.transcriptionConfidence`. The preset initializer gives no way to add the attribute. Change 004 uses the confidence to mark `word[?]`. Refs: REQ-TRN-001, REQ-TRN-004
- Feed the analyzer with `start(inputSequence:)` from an `AsyncStream<AnalyzerInput>` that a per-stream consumer task fills. The task reads chunks, trims samples before t0, converts with one long-lived `AVAudioConverter`, and yields the input. The converter keeps its state across chunks, so no click or gap appears at chunk edges. `AnalyzerInputConverter` needs macOS 27, so the tool does not use it. Refs: REQ-TRN-002, REQ-CAP-006, ADR-001
- Set `bufferStartTime` to `CMTime(value: hostDelta, timescale: analyzerSampleRate)` for every input, where `hostDelta` is chunk host time minus t0 in output samples. Results then come on the t0 timeline, and the tool adds no later offset. A start time on every input also keeps the time right after a filled gap. Refs: REQ-TRN-003, REQ-CAP-005, REQ-CAP-007
- Make a `TranscriptAssembler` per stream. It turns engine results into `TranscriptEvent.volatile(speaker, text, range)` and `TranscriptEvent.final(Segment)`. It trims text, drops empty results, and holds back a final result until no earlier volatile range is open, so finals leave in start order. It counts reorders for `debug-audio`. Refs: REQ-TRN-004, REQ-TRN-005
- `Segment` is a `Sendable` struct with `speaker`, `start`, `end` (seconds as `Double`), `text`, and `words: [Word]`, where a `Word` has `text` and `confidence: Double?`. Seconds as `Double` keep change 004 free of CoreMedia. Refs: REQ-TRN-004, ADR-006
- The terminal printer writes finals to stdout with one `fflush` per line. It draws the volatile line with `\r` and an erase sequence only when `isatty(1)` is true. A pipe in the gate scripts then gets clean lines. Refs: REQ-TRN-006
- Add `SpeechDetector(detectionOptions: .init(sensitivityLevel: .medium), reportResults: true)` to the Me analyzer. The assembler also drops a Me final whose whole range lies in a detector range with `speechDetected == false`. This guard holds even if the detector does not gate the transcriber by itself. The silence gate measures both effects. Them starts with no detector. TTS and call audio are speech most of the time, and the detector adds latency. The silence run on Them decides. Refs: REQ-TRN-007, ADR-003, ADR-008
- `doctor` treats the model as installed when `SpeechTranscriber.installedLocales` holds a locale equal to `en_US` by `supportedLocale(equivalentTo:)`. It also prints the `AssetInventory.status` value for the record. On the dev Mac, status said `supported` while the model worked, so status alone gives a false failure. The check calls no `assetInstallationRequest`, `reserve`, or `downloadAndInstall`. The model inventory sits behind a `ModelInventory` protocol so a test can count calls. Refs: REQ-TRN-008, ADR-005, ADR-007
- `record` runs the same check. If the model is missing, it calls `AssetInventory.assetInstallationRequest(supporting:)` and then `downloadAndInstall()`, and prints `progress.fractionCompleted` to stderr twice per second. A thrown error or a `nil` request with the model still missing exits 7. `record` does not call `reserve` unless a measurement shows the analyzer needs it. Refs: REQ-TRN-009, ADR-005, ADR-007
- A thrown analyzer error ends only that stream's engine. The consumer keeps reading chunks and drops them, so the capture ring buffer does not overflow. The warning maps `SFSpeechError.Code` values to plain text: `insufficientResources`, `tooManyAssetLocalesAllocated`, `noModel`, `audioDisordered`, and `unexpectedAudioFormat`. Other errors print their description. The warning goes into a shared `SessionWarnings` list for change 004. Refs: REQ-TRN-010, ADR-003
- Latency of a segment is the host clock at its arrival minus (t0 + segment end). When the env var `NOTETAKER_LATENCY_LOG` names a file, the tool appends one latency value per segment to that file. A gate script reads the file. A hidden option would break the rule that unknown options exit 1. Refs: REQ-TRN-012
- The recall fixture paragraph has no number words and no digits. The scorer lowers case and strips punctuation as the build plan says. The probe showed "three" becomes "3", so a number in the fixture would count as a miss with no real error. Refs: REQ-TRN-012, REQ-PRB-005
- The engine comparison uses `speech-probe` on recorded Me files from `say -o` played through the speakers into the microphone, and on one real read-aloud recording. The probe gains an `--engine dictation` flag that builds `DictationTranscriber` with the `timeIndexedLongDictation` preset plus `.punctuation` and the `.farField` hint. The audio files stay out of git. Refs: REQ-TRN-013, ADR-008

## Risks / Trade-offs

- Real call audio is harder than TTS. It has codec loss, cross-talk, and accents. The 90% recall on `say` is an upper bound. Mitigation: findings label TTS numbers as upper bounds, and the engine comparison adds one real recording.
- On laptop speakers the microphone hears Them, so Me can repeat Them lines. Mitigation: the M2 run with speakers counts these duplicates. The README says to use headphones until a measurement justifies duplicate removal.
- Product names, people's names, and jargon get wrong words. `SpeechTranscriber` takes no custom vocabulary. Mitigation: one task measures `contextualStrings` term recall and records it. No option ships in this change.
- The `installedLocales` check can disagree with `AssetInventory.status`. Either one can be wrong on a future OS. Mitigation: doctor prints both values, and findings record the case.
- Two analyzers at once can hit a resource limit on smaller Macs. Mitigation: the error becomes a warning and the other stream keeps going. The two-analyzer task records whether it happened on the dev Mac.
- A detector on Me can cut the first word of a sentence. Mitigation: the sentence gate checks the first word, and findings record any loss.
- `fastResults` can raise the share of volatile text that changes later. Mitigation: only finals reach the segment list.
