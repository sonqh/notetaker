# Tasks

## 1. Segment model and engine seam

- [ ] 1.1 Add `Speaker`, `Word`, `Segment`, and `TranscriptEvent` in `Sources/NotetakerKit/Transcription/`. Trim text, keep word confidences, and make no segment for empty text. [REQ-TRN-004] Verify: `swift test --filter SegmentTests`
- [ ] 1.2 Add the `TranscriptionEngine` protocol with plain result values, and a `FakeEngine` in the tests that plays scripted results and records each buffer start time. [REQ-TRN-011] Verify: `swift test --filter FakeEngineTests`
- [ ] 1.3 Add `TranscriptAssembler`: volatile replace, final clears volatile, start order, and the reorder count. Test it with the fake engine. [REQ-TRN-005, REQ-TRN-004] Verify: `swift test --filter TranscriptAssemblerTests`

## 2. Conversion and time mapping

- [ ] 2.1 Add the t0 mapping: trim samples before t0 and compute `bufferStartTime` in output samples from chunk host time. Cover a chunk after a filled gap. [REQ-TRN-003] Verify: `swift test --filter TimeMappingTests`
- [ ] 2.2 Add `ChunkConverter` with one long-lived `AVAudioConverter` per stream. Test 48 kHz stereo float to 16 kHz mono Int16 with a sine over ten chunks, and compare the sample count and edge continuity. [REQ-TRN-002] Verify: `swift test --filter ChunkConverterTests`
- [ ] 2.3 Add a source test that fails when a file under `Sources/` uses `AnalyzerInputConverter`, `ignoresResourceLimits`, or `CaptureInputSequenceProvider` outside an `#available` block. [REQ-TRN-002] Verify: `swift test --filter AvailabilityGuardTests`

## 3. Live engine and pipeline

- [ ] 3.1 Add `SpeechAnalyzerEngine`: build the transcriber from the preset options plus `.transcriptionConfidence`, start it with an `AsyncStream<AnalyzerInput>`, and map results to plain values. Add a test that writes speech with `say -o` to a temp file, runs it through the chunk path, and skips when `en_US` is not installed. [REQ-TRN-001, REQ-TRN-011] Verify: `swift test --filter SpeechAnalyzerEngineFileTests`
- [ ] 3.2 Add the speech gate on Me: a `SpeechDetector` module, and an assembler rule that drops a Me final inside a no-speech range. Test the rule with scripted detector results. [REQ-TRN-007] Verify: `swift test --filter SpeechGateTests`
- [ ] 3.3 Add `TranscriptionPipeline` per stream: a consumer task that reads chunks, converts, feeds the engine, and keeps draining chunks after an engine error. Test one failing stream and one live stream with fake engines. [REQ-TRN-010, REQ-TRN-001] Verify: `swift test --filter TranscriptionPipelineTests`
- [ ] 3.4 Map `SFSpeechError.Code` values to plain warning text with stream and time, and add each warning to `SessionWarnings`. [REQ-TRN-010] Verify: `swift test --filter EngineErrorMessageTests`

## 4. Speech model status and install

- [ ] 4.1 Add the `ModelInventory` protocol, the live inventory, and the doctor row. The row passes on `installedLocales` and also prints the `AssetInventory.status` value. A fake counts install, reserve, and download calls. [REQ-TRN-008] Verify: `swift test --filter ModelCheckTests`
- [ ] 4.2 Add the install path for `record`: install request, progress on stderr, and exit 7 on failure. Test it with the fake inventory. [REQ-TRN-009] Verify: `swift test --filter ModelInstallTests`
- [ ] 4.3 Run `doctor` on the dev Mac and record the model row, the status value, and the installed locales in `docs/findings.md`. Needs the dev Mac. [REQ-TRN-008] Verify: `scripts/gates/003-doctor-model.sh`

## 5. Terminal output and record wiring

- [ ] 5.1 Add `TerminalPrinter`: the `[HH:MM:SS] Speaker: text` line with a flush, the volatile line only on a terminal, and warnings on stderr. [REQ-TRN-006] Verify: `swift test --filter TerminalPrinterTests`
- [ ] 5.2 Wire `record` to start both pipelines after the model step, print through `TerminalPrinter`, and write the latency log when the env var `NOTETAKER_LATENCY_LOG` names a file. Record in findings whether two analyzers ran at once with no resource error. Needs the dev Mac with audio. [REQ-TRN-001, REQ-TRN-006, REQ-TRN-012] Verify: `scripts/gates/003-sentence.sh`

## 6. M2 gate measurements

- [ ] 6.1 Add a fixture paragraph of at least 150 words with no numbers in `Tests/Fixtures/`. Add `scripts/gates/003-recall.sh`: it plays the paragraph with `say -v Samantha` during `record`, scores the Them lines, and also scores a `say -o` file with `speech-probe`. Needs the dev Mac with audio. [REQ-TRN-012] Verify: `scripts/gates/003-recall.sh`
- [ ] 6.2 Add `scripts/gates/003-silence.sh`: 60 s of room silence, then print the Me and Them segment counts. Record the counts and the Them detector choice in findings. Needs the dev Mac with audio and a quiet room. [REQ-TRN-012, REQ-TRN-007] Verify: `scripts/gates/003-silence.sh`
- [ ] 6.3 Add `scripts/gates/003-latency.sh`: read the latency log of a paragraph run and print p50 and p95. Record them in findings. Needs the dev Mac with audio. [REQ-TRN-012] Verify: `scripts/gates/003-latency.sh`

## 7. Engine comparison and records

- [ ] 7.1 Add `--engine dictation` to `speech-probe`, with `.punctuation` and `.farField`. Add `scripts/gates/003-engines.sh`: it records `say` audio through the microphone and scores both engines on the same files. Needs the dev Mac with audio. [REQ-TRN-013] Verify: `scripts/gates/003-engines.sh`
- [ ] 7.2 Measure term recall with and without `contextualStrings` on a fixture full of product names, through `speech-probe`. Record the result in findings. Needs the en_US model. [REQ-TRN-012] Verify: `scripts/gates/003-context.sh`
- [ ] 7.3 Fill the M2 rows and the engine table in `docs/findings.md`, and record the engine choice per stream in ADR-008. Add a script that fails while an M2 row still says "not yet measured". [REQ-TRN-013, REQ-TRN-012, REQ-TRN-007] Verify: `scripts/gates/003-records.sh`
