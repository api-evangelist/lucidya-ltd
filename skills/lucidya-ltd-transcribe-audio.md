---
name: lucidya-transcribe-audio
description: >-
  Submit an audio file to Lucidya for offline transcription and speaker-diarized
  analysis, poll for completion, then retrieve the transcript with per-speaker
  dialect and theme classification. A three-call create-then-poll flow.
api: Lucidya AI API
base_url: https://api.lucidya.com
spec: ../openapi/lucidya-ltd-ai-api-openapi.yml
generated: '2026-08-13'
method: generated
source: openapi/lucidya-ltd-ai-api-openapi.yml
operations:
  - post_audio_transcription_transcribe_offline
  - checkAudioAnalysisStatus
  - getAudioAnalysisResult
---

# Transcribe and analyze audio

Asynchronous. You submit, you poll, you collect. Never expect the transcript on the
first response.

## 1. Submit the file

`post_audio_transcription_transcribe_offline` — `POST /audio_transcription/transcribe_offline`

- Header: `luc-authorization` (required)
- Query: `num_speakers` (integer, **required**) — the diarizer needs the speaker count
  up front; it is not inferred.
- Query: `language` (string, optional)
- Body: `multipart/form-data` with a `file` part (required) carrying the audio bytes.

Returns `AudioSubmissionResponse` → `{ "job_id": ..., "message": ... }`. Keep `job_id`.

Declared failures on submit: `400`, `401`, `429`.

## 2. Poll for status

`checkAudioAnalysisStatus` — `GET /audio_transcription/check_status?id=<job_id>`

- Query parameter is named `id`, not `job_id`.
- `200` → `AudioStatusResponse` `{ "job_id", "status" }`
- `202` → still running. Keep polling.
- `404` → unknown job id.

Treat `202` as "not done", never as success. Back off between polls — every poll spends
one of your **6 requests per minute** on the AI API.

## 3. Retrieve the result

`getAudioAnalysisResult` — `GET /audio_transcription/get_transcription_result?id=<job_id>`

- `200` → `AudioAnalysisResult` `{ "job_id", "status", "result" }`
- `202` → `AudioAnalysisInProgress`, same shape, result not yet populated. Go back to
  polling; do not parse `result`.
- `400`, `404` → bad or unknown id.

### Result structure

`result` is a list of `TranscriptSegment`:

| Field | Meaning |
|---|---|
| `text` | The transcribed utterance |
| `speaker` | Speaker label from diarization |
| `begins` | Segment start offset |
| `ends` | Segment end offset |

Per speaker, Lucidya also attaches `SpeakerDialect` (`dialect`, `sub_dialect`) and
`SpeakerThemes` (`themes`, `sub_themes`).

## Quota

Audio is metered by **duration**, not by request count: the total duration of one or
several audio files must not exceed **100 minutes of audio per minute**. That quota is
separate from the 6 requests/minute ceiling, and both apply at once.

## Rules

- No idempotency key exists. Re-submitting the same file after a timeout creates a
  **second** job and consumes quota twice. Persist the `job_id` before retrying.
- Poll the status endpoint, not the result endpoint, while waiting — both return 202,
  but status is the one documented for the purpose.
