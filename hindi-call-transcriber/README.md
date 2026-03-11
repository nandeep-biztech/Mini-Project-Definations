# Hindi Call Transcriber (With Speaker Diarization)

---

## 1️⃣ Core Objective

Build a production-style audio transcription platform that:

- Accepts Hindi call recording uploads (MP3/WAV)
- Preprocesses audio (mono, sample rate, normalization)
- Transcribes Hindi speech to English text (Whisper)
- Identifies speakers (who spoke when) via diarization
- Exposes transcripts in a web UI with search and export
- Runs transcription in a background job queue
- Tracks job status and progress end-to-end

This is not a simple "upload and transcribe" tool.  
It is a full-stack pipeline: upload → queue → worker → Python processing → storage → UI.

---

## 2️⃣ Tech Stack

- **Frontend**: Next.js (App Router), React, TailwindCSS, React Query
- **Backend API**: Node.js (Express.js), TypeScript
- **Job Queue**: BullMQ + Redis
- **Worker**: Node.js (queue consumer) + Python (transcription + diarization)
- **Transcription**: OpenAI Whisper (task=translate, Hindi → English)
- **Diarization**: pyannote.audio (optional)
- **Audio Processing**: ffmpeg / ffprobe
- **Database**: MongoDB
- **Monorepo**: pnpm workspaces + Turborepo
- **Shared**: TypeScript types and constants in a shared package

---

## 3️⃣ User Journey

### Flow

1. **Upload** — User lands on upload page. Drag-and-drop or select MP3/WAV file(s). Max file size (e.g. 100MB). Progress bar per file.
2. **Job Created** — On upload success, record is created in DB (status: PENDING). Job is enqueued (status: QUEUED). User is redirected to job detail page.
3. **Job Detail** — Page shows: filename, status badge, progress bar (0–100%), processing step (e.g. Transcribing → Diarizing → Merging). Auto-poll every 3 seconds until status is COMPLETED or FAILED.
4. **Jobs List** — Separate page listing all jobs with pagination. Columns: filename, status, progress, upload date. Click row → job detail. List auto-refreshes (e.g. every 5 seconds).
5. **Transcript View** — When job is COMPLETED, "View Transcript" is available. Transcript page shows:
   - Full transcript text
   - Speaker-colored segments with timestamps (start–end)
   - Speaker legend (SPEAKER_00, SPEAKER_01, …) with optional rename (e.g. SPEAKER_00 → "Amit")
   - Text search to filter segments
6. **Download** — User can download transcript as TXT, JSON, or SRT. Downloaded content must use custom speaker names if set.
7. **Failure** — If job fails, job detail shows error message. System should support retries (e.g. 2 attempts with backoff).

---

## 4️⃣ Upload & Storage

---

### A. File Acceptance

- Allowed types: MP3, WAV (validate MIME or extension)
- Max file size: configurable (e.g. 100MB)
- Reject oversize or invalid type with clear error

---

### B. Upload API

- **POST** `/api/upload`
- Content-Type: `multipart/form-data`, field name: `file`
- On success: save file to disk (e.g. `./data/uploads/`), create MongoDB record, enqueue BullMQ job
- Response: job id and initial status (PENDING → QUEUED)
- Extract duration using ffprobe before or after save (needed for UI)

---

### C. Storage Layout

- Base path configurable via env (e.g. `STORAGE_BASE_PATH=./data`)
- Subfolders: `uploads/` (raw files), optionally `processed/` (intermediate WAV), `transcripts/` (output files)
- API and Worker must resolve the same absolute path (same project root or config)

---

### D. Security & Rate Limiting

- Validate file type and size on server; never trust client only
- Rate limit upload endpoint (e.g. per IP or per user) to avoid abuse
- Do not apply aggressive rate limiting to polling endpoints (jobs list, job detail) to avoid breaking UI

---

## 5️⃣ Job Queue & Worker

---

### A. Queue Design

- Use BullMQ with Redis
- Single queue name (e.g. `transcription`)
- Job data: at least `audioFileId` (MongoDB document id) and path to uploaded file
- Job options: retries (e.g. 2), backoff (exponential)

---

### B. Node.js Worker Responsibilities

- Consume jobs from the queue
- For each job: resolve file path, spawn Python child process with script path and arguments (file path, output path, config flags)
- On Python exit: read result file (e.g. JSON), update MongoDB (transcript, speaker segments, status COMPLETED or FAILED)
- On Python error or timeout: mark job FAILED, store error message, optionally retry
- Graceful shutdown: finish current job, then stop consuming

---

### C. Python Script Responsibilities

Python must handle (in order):

1. **Preprocess** — Use ffmpeg: convert to mono, 16kHz (or Whisper-compatible format), loudness normalization. Do not over-trim silence (e.g. avoid silenceremove that can zero out short files).
2. **Transcribe** — Use OpenAI Whisper with `task=translate`, `language=hi` so Hindi is translated to English.
3. **Diarize** — Use pyannote.audio to get "who spoke when" segments (optional; can be disabled via env).
4. **Merge** — Align Whisper segments with diarization segments to produce `[{ speaker, start, end, text }]`.
5. **Output** — Write result to JSON file (and optionally TXT/SRT) for Node to read.

Python must:

- Use a virtual environment; Node must call the venv Python (e.g. `PYTHON_PATH`)
- Handle missing ffmpeg (clear error)
- Handle HuggingFace token for pyannote (gated model); if token missing or invalid, skip diarization or return empty segments with clear log

---

### D. Status and Progress

Intern must define a status enum and map progress to steps, e.g.:

- PENDING (0%)
- QUEUED (5%)
- TRANSCRIBING (10–50%)
- DIARIZING (50–75%)
- MERGING (75–90%)
- COMPLETED (100%) or FAILED (100%)

Worker updates `status` and `progress` in MongoDB as each step completes. Frontend polls job detail to show progress bar and step labels.

---

## 6️⃣ Database Schema

---

### A. Main Collection: `audiofiles` (or `AudioFile` model)

Intern must design schema to support:

| Field                 | Type     | Notes                                  |
| --------------------- | -------- | -------------------------------------- |
| id                    | ObjectId | PK                                     |
| originalFileName      | String   | Required                               |
| storagePath           | String   | Relative path under storage base       |
| mimeType              | String   | e.g. audio/mpeg, audio/wav              |
| fileSize              | Number   | Bytes                                  |
| duration              | Number   | Seconds (from ffprobe)                 |
| uploadDate            | Date     | Default now                            |
| status                | String   | Enum: pending, queued, transcribing, …  |
| progress              | Number   | 0–100                                  |
| error                 | String   | Null or error message if failed         |
| transcriptText        | String   | Full English transcript                |
| speakersTranscript    | Array    | [{ speaker, start, end, text }]        |
| speakerCount          | Number   | Number of distinct speakers            |
| speakerNames          | Map      | Optional: SPEAKER_00 → "Amit"          |
| processingStartedAt  | Date     | When worker started                    |
| processingCompletedAt | Date     | When worker finished                   |
| processingTimeMs      | Number   | Total processing time                  |

Add timestamps (createdAt, updatedAt) if using Mongoose.

---

### B. Indexes

- `status` (for jobs list filters)
- `uploadDate` (desc for "latest first")
- Optional: text index on `transcriptText` for search (can be done in API or frontend filter)

---

## 7️⃣ API Specification

---

### A. Endpoints (Summary)

| Method | Path                        | Description                              |
| ------ | --------------------------- | --------------------------------------- |
| POST   | /api/upload                 | Upload file (multipart), returns job id |
| GET    | /api/jobs                   | List jobs (?status=&page=&limit=)       |
| GET    | /api/jobs/:id               | Job detail (status, progress, error)    |
| GET    | /api/transcript/:id         | Full transcript + speaker segments      |
| PUT    | /api/transcript/:id/speakers| Rename speakers (body: speakerNames)     |
| GET    | /api/download/:id           | Download (?format=txt\|json\|srt)       |
| GET    | /health                     | Health check                             |

---

### B. Key Contracts

- **Upload response**: Include `id` and `status` so frontend can navigate to job detail.
- **Job detail**: Include `status`, `progress`, `error`, `transcriptText` (if completed), `speakersTranscript`, `speakerNames`.
- **Transcript**: Return 400 if still processing; 404 if not found.
- **Download**: Use `speakerNames` when generating TXT/SRT/JSON so custom names appear in exported file.

---

## 8️⃣ Speaker Rename Feature

- **Backend**: Add `speakerNames` field (e.g. Map&lt;String, String&gt;) to AudioFile. Endpoint **PUT** `/api/transcript/:id/speakers` with body `{ "speakerNames": { "SPEAKER_00": "Amit", "SPEAKER_01": "Priya" } }`. Persist and return updated document.
- **Frontend**: On transcript page, show speaker legend. Click speaker id → inline edit (input). On confirm, call API and update UI (optimistic update acceptable). Use updated names in segment list and in download.

---

## 9️⃣ UI Requirements (Next.js)

Must include:

- **Upload page** — Dropzone (e.g. react-dropzone), file list, upload button, progress bar per file. Redirect to job detail on success.
- **Jobs list page** — Table or cards: filename, status badge, progress bar, date. Pagination. Auto-refresh (e.g. 5s). Click → job detail.
- **Job detail page** — Filename, status, progress bar, step labels, timestamps (uploaded, started, completed), processing time. Error message if failed. "View Transcript" link when completed. Auto-poll every 3s until terminal status.
- **Transcript page** — Header (filename, speaker count), speaker legend with rename, search input (filter segments by text), list of segments (speaker color, timestamps, text). Download dropdown: TXT, JSON, SRT.
- **Loading and error states** — Skeletons or spinners while loading; clear error messages for failed upload or failed job.
- **LAN access** — App should work when opened from another device on same network (CORS, Next.js bind to 0.0.0.0, API client use same hostname).

---

## 🔟 Failure Handling Requirements

Intern must address:

- What if Python process crashes mid-run? (Mark job FAILED, store error, allow retry via BullMQ.)
- What if Redis or MongoDB is down during job processing? (Worker should fail job and optionally retry; health endpoint can check Redis/Mongo.)
- What if disk is full when saving uploaded file or transcript? (Return 500 on upload; mark job FAILED in worker with clear error.)
- How to avoid double-processing the same job? (Idempotent updates: set status to "processing" at start; only move to COMPLETED/FAILED once.)
- How to resume after worker restart? (BullMQ jobs remain in Redis; worker picks up again. Optionally mark "stuck" jobs after a timeout.)

Document decisions briefly in README or a short "Operations" section.

---

## 1️⃣1️⃣ Python Environment & External Tools

- **Python**: 3.9+ (3.9–3.11 recommended for pyannote). Use venv; Node calls `PYTHON_PATH` (e.g. `./workers/transcription-worker/.venv/bin/python3`).
- **ffmpeg / ffprobe**: Required. Document install (e.g. `brew install ffmpeg`). Worker or Python must find them (PATH or explicit path).
- **Whisper**: Install via pip. Model size configurable (e.g. WHISPER_MODEL=medium). First run downloads model.
- **pyannote.audio**: Gated model. User must accept terms on HuggingFace and set `HUGGINGFACE_TOKEN`. If token missing or pipeline fails, diarization can be skipped and segments returned empty.

---

## 1️⃣2️⃣ Non-Functional Requirements

- **Centralized config** — Environment variables for ports, MongoDB URI, Redis URL, storage path, Whisper model, diarization toggle, file size limit. No hardcoded secrets.
- **Structured logging** — Log with request id or job id where possible; levels (info, error). Prefer JSON logs in production.
- **Clean separation** — Controllers only handle HTTP; business logic in services. Worker: queue logic in Node, transcription logic in Python.
- **Shared types** — All DTOs and enums (e.g. ProcessingStatus, SpeakerSegment) in shared package. API, Worker, and Frontend import from shared.
- **No blocking** — Upload handler saves file and enqueues job; it does not run Whisper or Python. All heavy work in worker.
- **Git and build** — Proper .gitignore (node_modules, .env, data/uploads, venv). `pnpm build` builds shared then api/web/worker. Document how to run all services (e.g. 3 terminals or docker-compose).

---

## 1️⃣3️⃣ Acceptance Criteria (Checklist for Freshers)

The mini-project is complete when:

- [ ] User can upload MP3/WAV (up to configured max size) via drag-and-drop with progress bar.
- [ ] Jobs list shows all jobs with status and progress; list auto-refreshes; click goes to job detail.
- [ ] Job detail shows progress bar and step labels; auto-polls until COMPLETED or FAILED; shows error if failed.
- [ ] Hindi audio is transcribed to English text (Whisper, task=translate).
- [ ] Speaker diarization runs when enabled and produces speaker segments.
- [ ] Transcript page shows speaker-colored segments with timestamps.
- [ ] Speaker names can be renamed; custom names persist and appear in transcript and in download.
- [ ] Text search filters segments on transcript page.
- [ ] Download works for TXT, JSON, and SRT; exported content uses custom speaker names.
- [ ] Failed jobs show error message; worker retries up to N times with backoff.
- [ ] MongoDB and Redis can be started via Docker; app runs with API, frontend, and worker in separate processes.
- [ ] Shared package is used for types/constants across API, worker, and frontend.
- [ ] README explains setup (env, Python venv, ffmpeg, HuggingFace token), how to run each service, and how to open the app (including LAN).

---

*Use this document as the single source of truth for the Hindi Call Transcriber mini-project. Implement in the order that makes sense (e.g. monorepo → shared → API → worker → Python → frontend), and ensure all acceptance criteria are met before considering the project complete.*
