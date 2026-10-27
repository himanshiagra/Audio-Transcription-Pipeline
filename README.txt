1. Timestamped Audio Transcription API

We are using Python/FastAPI service using Faster-Whisper for converting speech-to-text. It accepts the audio uploads, stores the source, starts a background job, and exposes a JSON transcript with segment timestamps these are certain features that it does.

2.Run
Requires Python 3.11+, FFmpeg, and sufficient memory/CPU. While First transcription may download the model.

```bash
python -m venv .venv
# Windows: .venv\\Scripts\\activate
# macOS/Linux: source .venv/bin/activate
pip install -r requirements.txt
uvicorn app.main:app --reload
```
Open `/docs` for interactive API docs. Optional environment variables: `WHISPER_MODEL=base`, `WHISPER_DEVICE=cpu`, `WHISPER_COMPUTE_TYPE=int8`, `MAX_UPLOAD_BYTES=104857600`.

3. The Endpoints
- `POST /v1/transcriptions` — multipart upload with field `file`; returns 202 and a job ID.
- `GET /v1/transcriptions/{job_id}` — poll for queued, processing, completed, or failed state.
- `GET /health` — health check.

Example:
```bash
curl -X POST http://127.0.0.1:8000/v1/transcriptions -F "file=@sample.wav"
```

A completed response includes `text`, detected `language`, and `segments` containing `start` and `end` times in seconds and `text`.

4. How taking Engineering decisions
*Formats:*Certain Extension allow-list are that supports WAV, MP3, OGG, FLAC. Faster-Whisper uses FFmpeg for common container/codec decoding. The Production should validate file signatures and confirm the file can be decoded, not trust extensions alone.

*Long audio:* Whisper can process long recordings and returns timestamped segments. In very long media at scale, split into bounded chunks with a small overlap, offset each segment timestamp by its chunk's start time, and deduplicate text at chunk boundaries.

*Concurrent uploads:* Stream to disk in 1 MiB chunks to avoid holding entire files in RAM. Assign UUID job IDs and return immediately. For production, use object storage plus a durable queue (Celery/RQ/SQS) and a bounded worker pool sized to CPU/GPU memory. Add queue limits, back-pressure, per-user quotas and idempotency keys.

*Storage:* Demo uses `data/audio` and `data/results`. Production should store audio in private encrypted object storage and job state/transcript in a database or object store, with authorization, retention, and deletion policies.

*Retries:* Demo records failed status. Production should retry transient failures with exponential backoff and jitter, cap attempts, send exhausted jobs to a dead-letter queue, and make jobs idempotent.

*Observability:* Add auth, TLS, rate limits, structured logs and metrics for queue depth, duration, failure rate, retries and audio minutes. Never log raw audio or full transcript text unnecessarily.

5.The Production architecture
Client -> authenticated API -> object storage -> durable queue -> worker pool -> results database/object storage -> polling endpoint or webhook.

6. Some of the Limitations
The Local disk is not shared across replicas and the FastAPI background tasks are not durable and do not automatically retry. Authentication, cloud storage, durable queue, automatic retries, webhooks and load testing are production follow-ups.
