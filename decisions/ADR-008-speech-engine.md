# ADR-008 SpeechAnalyzer with SpeechTranscriber does the transcription

Status: accepted. Date: 2026-11-03.

## Decision

- Each stream gets one `SpeechAnalyzer` with a `SpeechTranscriber` for `en_US`.
- The transcriber uses the preset `timeIndexedProgressiveTranscription`: volatile results, fast results, and audio time ranges.
- Each input buffer carries `bufferStartTime` on the t0 timeline.
- `SpeechDetector` gates the Me stream.
- M2 measures `DictationTranscriber` with punctuation and `farField` on the Me stream. The better engine wins, and this record changes.
- The code uses no API that needs macOS 27, such as `AnalyzerInputConverter`, unless `#available` guards it.

## Why

- The framework runs on device and needs no third-party model.
- `SpeechTranscriber` has no punctuation option and no content hints. `DictationTranscriber` has both, so it is the fallback to measure.

## Consequences

- Speech from a real call is harder than text-to-speech audio. Quality numbers from `say` are upper bounds.

Refs: REQ-TRN-001, REQ-TRN-007, REQ-TRN-013
