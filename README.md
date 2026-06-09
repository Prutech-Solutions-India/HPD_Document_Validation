# HPD Document Verification — Architecture & Operations


## 1. Executive Summary

HPD is a two-tier document-verification stack:

- **Web-UI** — Next.js 14 app (auth, dashboards, applicant/case CRUD, file storage). Owns all persistence.
- **Web-api** — Stateless FastAPI service that runs the verification pipeline (OCR → LLM → rules → field-match → score) and returns a decision.

The verifier runs fully on-prem against a local **Ollama** model (`llama3.1:8b`) and **PaddleOCR**, so there is **no per-document SaaS cost**. The same `OLLAMA_URL` knob can point at any OpenAI-compatible gateway when self-hosting is outgrown.

---

## 2. Component Diagram

```mermaid
flowchart LR
  User[Browser] -->|HTTPS| UI[Web-UI<br/>Next.js 14 + Prisma]
  UI -->|Postgres wire| DB[(Postgres<br/>applicants, cases,<br/>documents, runs, users)]
  UI -->|local FS| Disk[(uploads/<br/>file blobs)]
  UI -->|Bearer token<br/>multipart POST| API[Web-api<br/>FastAPI + Uvicorn]
  API -->|HTTP /api/generate| OLM[Ollama<br/>llama3.1:8b]
  API -->|in-proc| OCR[PaddleOCR<br/>+ pypdfium2 / python-docx]
  API -->|JSON files| Audit[(audit/<br/>YYYY-MM-DD/run_id.json)]
```

### Components

| Component | Tech                                             | Role                                                   | State                            |
| --------- | ------------------------------------------------ | ------------------------------------------------------ | -------------------------------- |
| Web-UI    | Next.js 14, React 19, Tailwind, Prisma, Postgres | User-facing pages, auth, persistence, file storage     | Stateful (Postgres + `uploads/`) |
| Web-api   | FastAPI, Uvicorn, Pydantic v2                    | Pure verification engine                               | **Stateless**                    |
| Ollama    | `llama3.1:8b` (4.7 GB)                           | Field extraction from OCR text                         | Model weights on disk            |
| Postgres  | PG 14+                                           | Applicants, cases, documents, verification runs, users | Persistent                       |

---

## 3. Request Flow — Single Document Upload

```mermaid
sequenceDiagram
  participant B as Browser
  participant UI as Next.js (Web-UI)
  participant DB as Postgres
  participant FS as uploads/
  participant API as Web-api
  participant LLM as Ollama
  participant AUD as audit/

  B->>UI: POST /api/upload (multipart)
  UI->>UI: requireAuth() · pickDocumentType()
  UI->>DB: find Applicant · get-or-create VerificationCase
  UI->>API: POST /api/pipeline/run<br/>Authorization: Bearer …
  API->>API: RequestId · size-cap · CORS · auth · magic-byte sniff
  API->>API: OCR (pypdfium / docx / Paddle)
  API->>LLM: /api/generate (format=json)
  LLM-->>API: extracted fields (JSON)
  API->>API: rules · field_match · score
  API->>AUD: write {run_id}.json
  API-->>UI: PipelineRunResponse
  UI->>DB: create Document · roll up VerificationCase
  UI->>FS: write uploads/{doc_id}
  UI->>UI: kickUploadSweep() (rate-limited)
  UI-->>B: 200 + decision payload
```

### Pipeline stages ([Web-api/app/pipeline/runner.py](Web-api/app/pipeline/runner.py))

1. **OCR** — `pypdfium2` text layer first, fall back to PaddleOCR rasterise. DOCX → `python-docx`. XLSX → `openpyxl`. Images → PaddleOCR.
2. **LLM** — Ollama `/api/generate` with `format=json`, `temperature=0`. One retry on malformed JSON.
3. **Rules** — expiry windows (90 days for bank statements, etc.), minimum field presence, type-specific guards.
4. **Field match** — applicant-claimed values vs extracted values via RapidFuzz; name aliases, DOB format normalisation.
5. **Score** — risk thresholds drive final status: `PASSED` / `FAILED` / `MANUAL_REVIEW` / `UNREADABLE` / `EXPIRED` / `DATA_MISMATCH`.

### Why OCR _and_ an LLM? (both are required)

The two stages do different jobs and neither can replace the other:

| Stage                                         | Input it accepts                                                    | Output it produces                                           | Why we need it                                                                                                                                                                                                                                                       |
| --------------------------------------------- | ------------------------------------------------------------------- | ------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **OCR** (PaddleOCR / pypdfium2 / python-docx) | Pixels, scanned PDFs, photos of IDs, faxes, image-only Office files | Raw UTF-8 text + per-line confidence                         | `llama3.1:8b` is a **text-only** model. Without OCR, scanned documents and ID photos arrive as opaque bytes and the LLM has nothing to read.                                                                                                                         |
| **LLM** (Ollama `llama3.1:8b`)                | Raw OCR text (often noisy, multi-column, mis-ordered)               | Structured `{field: value}` JSON with semantic understanding | Regex / keyword extraction breaks on the layout variance of real-world IDs, bank statements and utility bills. The LLM normalises wording ("D.O.B" vs "Date of Birth" vs "Born"), handles multilingual hints, and emits a stable schema the rules engine can act on. |

In short:

- **OCR turns paper / pixels into text.** Skip it → image uploads return `UNREADABLE`.
- **The LLM turns messy text into trustworthy fields.** Skip it → a hand-written parser would be needed per document type, per layout, per locale — unmaintainable.

PDFs with a native text layer skip PaddleOCR (we read the layer directly), which is why DOCX / XLSX / digital PDFs are ~10× faster than scanned ones.

---

## 4. Infrastructure

### Minimum viable — single host docker-compose (shipped)

| Service                     | CPU          | RAM                   | Disk                                                             | Notes                                                           |
| --------------------------- | ------------ | --------------------- | ---------------------------------------------------------------- | --------------------------------------------------------------- |
| `ui` (Next.js)              | 1 vCPU       | 512 MB                | 1 GB image                                                       | Port 80                                                         |
| `api` (Uvicorn × 2 workers) | 2 vCPU       | 1 GB                  | 2 GB image                                                       | Internal-only                                                   |
| `ollama`                    | **4–8 vCPU** | **8 GB** for 8B model | 5 GB model + cache                                               | Internal-only                                                   |
| `db` (Postgres 16)          | 1 vCPU       | 1 GB                  | 10 GB                                                            | Bundled; swap to managed RDS / Aurora by setting `DATABASE_URL` |
| Volumes                     | –            | –                     | `uploads` ~ N × 2 MB · `audit` ~ 5 KB/run · `ollama-models` 5 GB |                                                                 |

**Recommended host: 8 vCPU / 16 GB RAM / 100 GB SSD, Linux + Docker Engine 24+. No GPU required.**

### Production-grade — horizontal scale

- **UI** behind nginx / Traefik + Let's Encrypt → 2…N replicas (stateless app, sessions in cookie).
- **api** → 2…N replicas (stateless after commit `27213a8`).
- **ollama** → 1× GPU node (T4 / A10 / L4) gives **10–15× speedup**; or swap to a managed LLM by changing `OLLAMA_URL`.
- **Postgres** → managed (RDS / Cloud SQL / Aurora Serverless).
- **uploads** → migrate to S3 / Blob storage; today is a local volume swept after `UPLOAD_RETENTION_DAYS=30`.
- **audit** → ship to Loki / CloudWatch / Splunk.

### Network topology (compose)

```mermaid
flowchart TB
  internet((Internet)) -->|:80| ui
  subgraph docker[Docker bridge network]
    ui[hpd-ui<br/>Next.js :3000]
    api[hpd-api<br/>Uvicorn :8000]
    ol[hpd-ollama<br/>:11434]
  end
  ui -->|Bearer auth| api
  api --> ol
  ui -.->|wire protocol| pg[(Postgres<br/>external)]
```

---

## 5. Cost Model (USD/month, illustrative)

### Self-hosted CPU only — small org (≤ 5 000 docs/month)

| Item                                      | Spec                                                              | $/month        |
| ----------------------------------------- | ----------------------------------------------------------------- | -------------- |
| 1× VM                                     | 8 vCPU / 16 GB / 100 GB (Hetzner CCX23, Linode 16 GB, OCI Ampere) | **30 – 80**    |
| Managed Postgres                          | 1 vCPU / 2 GB                                                     | 25 – 50        |
| Object storage (when uploads moves to S3) | 50 GB + egress                                                    | 5 – 15         |
| Backups (snapshots)                       | –                                                                 | 5 – 10         |
| **Total**                                 |                                                                   | **~ 65 – 155** |

LLM cost: **$0** (Ollama runs locally).

### Self-hosted with GPU — mid (50 000 docs/month)

| Item            | Spec                         | $/month         |
| --------------- | ---------------------------- | --------------- |
| GPU VM (Ollama) | 8 vCPU / 32 GB / 1× T4 or L4 | **300 – 600**   |
| API/UI VM       | 4 vCPU / 8 GB                | 30 – 60         |
| Postgres        | 2 vCPU / 4 GB                | 60 – 120        |
| S3-equivalent   | 200 GB                       | 10 – 20         |
| **Total**       |                              | **~ 400 – 800** |

### Cloud LLM swap (when self-hosting is outgrown)

Set `OLLAMA_URL` to a compatible gateway — no code change. Typical doc is ~2–4 K input + 500 output tokens.

| Provider / model         | ~$ per 1 000 docs |
| ------------------------ | ----------------- |
| OpenAI `gpt-4o-mini`     | 3 – 5             |
| Anthropic `claude-haiku` | 3 – 5             |
| Groq `llama-3.1-8b`      | 0.1 – 0.3         |
| Self-hosted              | 0                 |

### What is **not** billed

- PaddleOCR — Apache 2.0
- Llama 3.1 — Meta community licence (≤ 700 M MAU)
- Per-seat UI cost

---

## 6. Multiprocessing & Concurrency

### Baseline: how long does one file take?

Measured on a Windows dev box, **CPU only**, `llama3.1:8b`, PaddleOCR warm. These are the times one isolated request takes — concurrency math below builds on top of these numbers.

| File type / shape           | OCR stage                          | LLM stage           | Rules + match + score | **Total / file**            |
| --------------------------- | ---------------------------------- | ------------------- | --------------------- | --------------------------- |
| DOCX / XLSX (native text)   | < 100 ms                           | 15 – 35 s           | < 50 ms               | **~15 – 35 s**              |
| PDF with embedded text      | 200 – 500 ms                       | 15 – 35 s           | < 50 ms               | **~15 – 35 s**              |
| PDF scanned / photo / image | 8 – 20 s (PaddleOCR)               | 15 – 35 s           | < 50 ms               | **~25 – 55 s**              |
| First request after boot    | +20 – 40 s (Paddle model download) | +30 s (Ollama warm) | –                     | **+60 – 90 s, one-time**    |
| With a GPU (T4 / L4)        | same as above                      | **2 – 5 s**         | < 50 ms               | **~3 – 6 s** (digital docs) |

Rule of thumb: **~20 s per digital file, ~40 s per scanned file** on the default CPU box. A GPU collapses the LLM step ~10× and is the single biggest lever.

### How requests flow when many users upload at once

```mermaid
flowchart LR
  U1[10–20 users<br/>click Submit] --> UI[Next.js UI<br/>fully async, ~unlimited]
  UI --> ACC[Uvicorn socket<br/>accept queue]
  ACC --> W1[API worker 1<br/>1 req in flight]
  ACC --> W2[API worker 2<br/>1 req in flight]
  ACC -.queued.-> WAIT[(backlog)]
  W1 --> OL[Ollama<br/>CPU: 1 inference at a time<br/>GPU: 4–8 in parallel]
  W2 --> OL
```

**Bottom line:** every upload is accepted instantly. The number that are _actively crunching_ at any moment equals **`min(WEB_API_WORKERS, Ollama capacity)`**. Everything else waits politely in the socket backlog — no failures, no timeouts.

### Per-layer limits (today's defaults)

| Layer              | Mechanism                                                | Default                                                                     |
| ------------------ | -------------------------------------------------------- | --------------------------------------------------------------------------- |
| Web-UI (Next.js)   | Node async I/O                                           | ~unlimited (DB + forward only)                                              |
| Postgres           | Connection pool                                          | hundreds in parallel                                                        |
| Uvicorn workers    | OS processes, true parallel                              | `WEB_API_WORKERS=2` ([Dockerfile](Web-api/Dockerfile)) — set ≈ 1× CPU count |
| FastAPI per worker | async loop; sync calls (today's `httpx.Client`) block it | **1 in-flight per worker**                                                  |
| PaddleOCR          | NumPy / Paddle internal threads                          | 1 process / N threads                                                       |
| Ollama (CPU)       | **single inference at a time per model**                 | concurrency = 1                                                             |
| Ollama (GPU)       | KV-cache batching                                        | concurrency ≈ 4–8 on T4 / L4                                                |
| Upload sweeper     | rate-limited (1×/hour), non-blocking                     | [runs.ts](Web-UI/src/lib/db/runs.ts)                                        |

### Realistic concurrency today

| Setup                                   | Active in parallel | 10 users × 3 docs (30 docs) wall-clock |
| --------------------------------------- | ------------------ | -------------------------------------- |
| Default: 2 workers, CPU-only Ollama     | **1–2**            | ~12–13 min                             |
| Default + async httpx fix (roadmap #12) | 1–2                | ~12 min (still gated by Ollama)        |
| 8 workers, CPU-only Ollama              | **1**              | ~12 min — workers wait for Ollama      |
| 8 workers, **1× T4 GPU** Ollama         | **4–8**            | **~45 – 90 s**                         |
| 8 workers, 2× Ollama replicas (CPU)     | 2                  | ~6 min                                 |
| 8 workers, GPU + async httpx            | 4–8                | < 1 min                                |

### Why workers alone don't help on CPU-only Ollama

[Web-api/app/pipeline/llm.py](Web-api/app/pipeline/llm.py) uses a **synchronous** `httpx.Client`, which blocks the worker's event loop for the full 15–35 s of the LLM call. With the model running on CPU, Ollama itself can only process one inference at a time. So no matter how many workers we add, only one is ever doing useful LLM work — the rest are blocked waiting their turn at the same Ollama instance.

### Scaling roadmap — pick the level required

| User load (concurrent submitters) | Recommended setup                                                                                         | Effort                                |
| --------------------------------- | --------------------------------------------------------------------------------------------------------- | ------------------------------------- |
| 1 – 3                             | As-is: 2 workers, CPU Ollama                                                                              | Zero                                  |
| 5 – 10                            | `WEB_API_WORKERS=4` + **GPU** for Ollama (T4 / L4)                                                        | Hardware swap, no code change         |
| 10 – 30                           | `WEB_API_WORKERS=8` + GPU + **async httpx** (#12)                                                         | One file change in `llm.py`           |
| 30 – 100                          | Above + **2–4 Ollama replicas** behind a load balancer                                                    | Compose / k8s topology change         |
| 100+ (bursty)                     | Add **job queue** (Redis + RQ / Celery): UI returns 202, worker pool processes, user polls / gets webhook | Architectural shift — moderate effort |
| 1 000+ sustained                  | Multi-region: GPU Ollama pool + autoscaling API pool + S3 uploads + read-replica Postgres                 | Cloud-native deployment, infra heavy  |

### Hardening that pairs with scale-out

- **Rate-limit** `/api/pipeline/run` per token / IP (roadmap #17) — keeps one buggy client from starving the queue.
- **Per-request timeout** already enforced (`OLLAMA_TIMEOUT_S`, default 180 s) so a hung inference can't pin a worker forever.
- **Healthchecks** drop unhealthy API / Ollama replicas from the LB automatically (compose `depends_on: condition: service_healthy` today; same hook in k8s readiness probes).
- **Audit log** is per-request and lock-free, so it does not bottleneck under load.

### TL;DR

- **Single user, single document?** Works in 15–35 s.
- **Single user, multiple documents?** Today they process sequentially (only 1–2 workers, Ollama serialises).
- **Multiple users at once?** All accepted; ~2 actually crunching, rest queue.
- **Want true 10–20 parallel?** Add a GPU and bump `WEB_API_WORKERS` — no code change to the pipeline.

---

## 7. Latency & Throughput (observed)

Measured on a Windows dev box, **CPU only**, `llama3.1:8b`, PaddleOCR warm.

### Per document

| Document type             | OCR                                | LLM                 | Rules + match + score | **Total/doc**           |
| ------------------------- | ---------------------------------- | ------------------- | --------------------- | ----------------------- |
| DOCX / XLSX (native text) | < 100 ms                           | 15 – 35 s           | < 50 ms               | **15 – 35 s**           |
| PDF with text layer       | 200 – 500 ms                       | 15 – 35 s           | < 50 ms               | **15 – 35 s**           |
| PDF scanned / image       | 8 – 20 s (Paddle)                  | 15 – 35 s           | < 50 ms               | **25 – 55 s**           |
| First request after boot  | +20 – 40 s (Paddle model download) | +30 s (Ollama warm) | –                     | **+60 – 90 s one-time** |

### Per applicant (typical 3 documents: ID + address + income)

| Hardware           | Sequential | With 2 API workers (CPU)         | With GPU (T4) |
| ------------------ | ---------- | -------------------------------- | ------------- |
| 8 vCPU CPU only    | 60 – 120 s | ~ 60 – 120 s (Ollama serialises) | –             |
| 8 vCPU + 1× T4 GPU | 6 – 15 s   | 4 – 8 s                          | **4 – 8 s**   |
| Stub mode (CI)     | < 0.5 s    | < 0.5 s                          | < 0.5 s       |

### Sustained throughput (peak)

| Setup                      | Docs/hour        |
| -------------------------- | ---------------- |
| CPU only                   | 30 – 60          |
| 1× T4 GPU                  | 400 – 900        |
| Stub mode (CI / load test) | 5 000 per worker |

---

## 8. Security Controls (in code)

| Control                  | Where                                                                                                               | Default                                   |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------- | ----------------------------------------- |
| Bearer-token auth        | `Authorization: Bearer $WEB_API_TOKEN` on every `/api/*` route ([Web-api/app/security.py](Web-api/app/security.py)) | Empty = dev only                          |
| Request-size cap         | `MaxBodySizeMiddleware` → HTTP 413                                                                                  | 25 MB (`MAX_UPLOAD_BYTES`)                |
| Magic-byte sniffer       | `assert_allowed_upload()` rejects ext/content mismatch → HTTP 415                                                   | PDF, DOCX, XLSX, JPG, PNG, GIF, TIFF, TXT |
| CORS                     | `CORS_ORIGINS` env, comma-separated                                                                                 | `http://localhost:3000`                   |
| Request-Id propagation   | `X-Request-Id` header echoed; logged on every line                                                                  | uuid4 if absent                           |
| Audit log                | JSON per `run_id` in `AUDIT_DIR/YYYY-MM-DD/`                                                                        | `/var/lib/hpd/audit` in container         |
| Upload retention         | hourly sweep removes orphan/aged files in `uploads/` ([Web-UI/src/lib/db/runs.ts](Web-UI/src/lib/db/runs.ts))       | 30 days (`UPLOAD_RETENTION_DAYS`)         |
| Reverse-proxy headers    | `--proxy-headers --forwarded-allow-ips='*'`                                                                         | on                                        |
| Compose secret fail-fast | `${WEB_API_TOKEN:?…}` blocks boot if unset                                                                          | enforced                                  |

---

## 9. API Surface

The stack exposes two HTTP surfaces:

- **Web-UI (Next.js, port 80)** — owns auth, applicants, applications, documents, validation rules. All browser traffic terminates here.
- **Web-api (FastAPI, port 81)** — pure verification engine. Token-gated. Called only by the UI (server-to-server).

### 9.1 Web-api — verifier (FastAPI)

| Method | Path                | Auth         | Description                                  |
| ------ | ------------------- | ------------ | -------------------------------------------- |
| GET    | `/livez`            | no           | Liveness — process up, no external calls     |
| GET    | `/readyz`           | no           | Readiness — Ollama reachable + mode snapshot |
| GET    | `/health`           | no           | Back-compat alias for `/readyz`              |
| POST   | `/api/pipeline/run` | Bearer token | Multipart upload; returns decision           |

### 9.2 Web-UI — application API (Next.js Route Handlers)

All routes are session-cookie-authenticated (`requireAuth()`) unless flagged otherwise. Source under [Web-UI/src/app/api/](Web-UI/src/app/api/).

| Method   | Path                               | Description                                                                                    |
| -------- | ---------------------------------- | ---------------------------------------------------------------------------------------------- |
| `POST`   | `/api/auth/login`                  | **Public.** Email + password → sets JWT session cookie                                         |
| `POST`   | `/api/auth/logout`                 | Clears session cookie                                                                          |
| `GET`    | `/api/auth/me`                     | Current reviewer profile from the cookie                                                       |
| `GET`    | `/api/applicants`                  | List applicants (master record, seeded externally)                                             |
| `POST`   | `/api/upload`                      | Multipart upload → writes to S3, calls verifier, persists Document + rolls up case             |
| `GET`    | `/api/applications/[id]`           | Application detail (case + applicant + documents + runs)                                       |
| `POST`   | `/api/applications/[id]/runs`      | Re-run verification on the existing documents in the case                                      |
| `GET`    | `/api/applications/[id]/runs`      | List verification runs for the application                                                     |
| `POST`   | `/api/applications/[id]/verify`    | Finalise/lock a verified application                                                           |
| `GET`    | `/api/documents/[id]/file`         | Stream the stored bytes from S3 (Open / Download buttons)                                      |
| `DELETE` | `/api/documents/[id]`              | Soft-delete a document from the case                                                           |
| `GET`    | `/api/document-types`              | List supported document types (35 NYC HPD types)                                               |
| `POST`   | `/api/document-types`              | Create a new document type (admin)                                                             |
| `PATCH`  | `/api/document-types/[id]`         | Rename / re-activate / deactivate                                                              |
| `DELETE` | `/api/document-types/[id]`         | Hard-delete a document type (admin)                                                            |
| `GET`    | `/api/document-field-checks`       | List validation rules grouped by document type                                                 |
| `POST`   | `/api/document-field-checks`       | Create a validation rule (check_type ∈ extract/presence/not_expired/recent_within/cross_check) |
| `PATCH`  | `/api/document-field-checks/[id]`  | Update a rule (active flag, kind, days, attr, …)                                               |
| `DELETE` | `/api/document-field-checks/[id]`  | Remove a rule                                                                                  |
| `GET`    | `/api/manual-review`               | Queue of cases needing reviewer attention                                                      |
| `POST`   | `/api/manual-review/[id]/decision` | Reviewer decision: `approve` / `reject` with reason                                            |

### 9.3 Deployed instance & host ports

The single-host docker-compose stack publishes each container on a dedicated host port. Defaults below; all are overridable in `.env`.

| Service         | Container port | Host port | Live URL (POC server `52.66.180.185`)                             | `.env` knob        |
| --------------- | -------------- | --------- | ----------------------------------------------------------------- | ------------------ |
| `ui` (Next.js)  | `3000`         | **80**    | http://52.66.180.185/login                                        | (fixed at 80)      |
| `api` (FastAPI) | `8000`         | **81**    | http://52.66.180.185:81/livez                                     | `API_HOST_PORT`    |
| `db` (Postgres) | `5432`         | **5432**  | `postgresql://postgres:postgres@52.66.180.185:5432/doc_verify_db` | `DB_HOST_PORT`     |
| `ollama`        | `11434`        | **90**    | http://52.66.180.185:90/api/tags                                  | `OLLAMA_HOST_PORT` |

**Reachable from the server / monitoring:**

```bash
# UI
curl -I http://52.66.180.185/login

# API health (no auth required)
curl http://52.66.180.185:81/livez       # {"status":"alive"}
curl http://52.66.180.185:81/readyz      # {"status":"ready","llm_health":"ok",...}
curl http://52.66.180.185:81/health      # back-compat alias for /readyz

# Verifier (POST, token-gated, rate-limited — UI calls this server-side only)
curl -X POST http://52.66.180.185:81/api/pipeline/run \
  -H "Authorization: Bearer $WEB_API_TOKEN" \
  -F "file=@./passport.pdf" \
  -F "document_type=passport" \
  -F "applicant_name=Jane Doe"

# DB (PG 16, requires creds)
psql "postgresql://postgres:postgres@52.66.180.185:5432/doc_verify_db" -c "select count(*) from \"Applicant\";"

# Ollama (model registry — confirms llama3.1:8b is pulled)
curl http://52.66.180.185:90/api/tags
```

### 9.4 Demo credentials & end-to-end login + upload

> **POC only.** Rotate the password and the `WEB_API_TOKEN` before any external demo.

| What               | Value                                                                              |
| ------------------ | ---------------------------------------------------------------------------------- |
| Login URL          | http://52.66.180.185/login                                                         |
| Reviewer email     | `subhadeep.roy@prutech.com`                                                        |
| Reviewer password  | `password123`                                                                      |
| Seeded applicants  | 10 (Priya Natarajan, Arjun Mehta, Ananya Iyer, Rohan Banerjee, Sneha Kapoor, …)    |
| Test applicant ref | use any `reference` from the Applicants page (e.g. the first row in the dashboard) |
| Default doc bucket | S3 `devteam-cms` / prefix `uploads/` in `ap-south-1`                               |

Source of truth: [Web-UI/prisma/seed.ts](Web-UI/prisma/seed.ts) (`DEFAULT_USER_EMAIL` / `DEFAULT_USER_PASSWORD`) and [Web-UI/prisma/ensure-user.ts](Web-UI/prisma/ensure-user.ts) (recreates the reviewer if missing on boot).

#### Browser flow (the way real users hit it)

1. Open http://52.66.180.185/login.
2. Sign in with `subhadeep.roy@prutech.com` / `password123`. The server sets an HTTP-only JWT cookie (`session` / `app_session`, depending on env).
3. Navigate to **Applications** → pick (or create) an application for a seeded applicant.
4. Click **Upload Document**, pick a `.pdf` / `.docx` / `.jpg` / `.png` (up to `MAX_UPLOAD_BYTES`, default 25 MB), choose a `document_type`. Multiple files upload in parallel.
5. The case auto-rolls up to `PASSED` / `FAILED` / `MANUAL_REVIEW` / `DATA_MISMATCH` based on the verifier response.
6. **Open** / **Download** on a document row streams the bytes back from S3 via `/api/documents/{id}/file`.

#### Script flow (login + upload via curl)

```bash
# 1. Login → save the JWT cookie
curl -s -c /tmp/hpd.cookies \
  -H "Content-Type: application/json" \
  -X POST http://52.66.180.185/api/auth/login \
  -d '{"email":"subhadeep.roy@prutech.com","password":"password123"}'

# 2. Confirm session
curl -s -b /tmp/hpd.cookies http://52.66.180.185/api/auth/me

# 3. List applicants → grab a reference
curl -s -b /tmp/hpd.cookies http://52.66.180.185/api/applicants | jq '.[0].reference'

# 4. Upload a document (multipart; field name `application_id` carries the
#    Applicant.reference for backward compatibility)
curl -s -b /tmp/hpd.cookies \
  -X POST http://52.66.180.185/api/upload \
  -F "file=@./Sample Files/passport.pdf" \
  -F "application_id=APP-001" \
  -F "document_type=passport"
```

**Security note:** all four ports default to `0.0.0.0`. For anything beyond a POC, restrict `:5432` and `:90` (Ollama) to the LAN / monitoring subnet at the firewall — only `:80` (UI) and `:81` (API health + token-gated pipeline) need public exposure. Note that `:90` is a privileged port (<1024), so the Docker daemon must run as root to bind it. TLS termination in front of `:80` / `:81` is roadmap item #TLS.

### Supported `document_type` values

The verifier knows **35 NYC HPD document types** (ported from the legacy .NET `DocumentType` enum). Each has a dedicated field-extraction spec in [`FIELD_SPECS`](Web-api/app/pipeline/prompts.py) and a rule set in [`RULES`](Web-api/app/pipeline/rules.py). Anything outside the list silently degrades to `other`.

| Category              | `document_type` values                                                                                                                                                                                  |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Identity              | `passport`, `drivers_license`, `national_id`, `picture_id`, `military_id`, `ssn_card`, `birth_certificate`, `marriage_certificate`, `declaration_of_emancipation`, `legal_custody`, `school_enrollment` |
| Employment / income   | `pay_stub`, `w2`, `tax_return`, `form_1040`, `form_1099`, `cash_payments`, `self_employment_income`                                                                                                     |
| Government / benefits | `social_security_award_letter`, `ssi_ssdi`, `veterans_benefits`, `pa_budget_letter`, `armed_forces_reserves`, `pension_letter`, `unemployment`, `disability_certificate`                                |
| Other income          | `child_support_alimony`, `dividends_annuity`, `rental_income`, `gift_income`                                                                                                                            |
| Assets / banking      | `bank_statement`, `investment_account`, `real_estate_statement`                                                                                                                                         |
| Housing               | `nycha_lease`, `section_8_proof`                                                                                                                                                                        |
| Misc                  | `utility_bill`, `employment_letter`, `income_certificate`, `other`                                                                                                                                      |

Recency windows match the HPD reference: pay stubs / W-2s / SSA / SSI / VA / PA / pension / Section 8 within **120 days**, bank statements within **90 days**, employment letters / income certificates within **365 days**.

---

## 10. Roadmap (what's not done yet, prioritised)

| #     | Item                                                                    | Why it matters                     |
| ----- | ----------------------------------------------------------------------- | ---------------------------------- |
| ✅ 17 | Rate limit `/api/pipeline/run` (in-process token bucket per token / IP) | One client can pin Ollama          |
| ✅ 12 | Long-lived `httpx.AsyncClient` to Ollama                                | Saves TCP setup, frees worker loop |
| ✅ 13 | CI (GitHub Actions: pytest + tsc)                                       | Catch regressions                  |
| –     | TLS / reverse proxy in front of UI                                      | HTTP only today                    |
| –     | Object storage for `uploads/`                                           | Outgrows single host               |
| 11    | PaddleOCR warmup log line at startup                                    | Surfaces first-request latency     |
| 14    | Real per-line OCR confidence                                            | Currently flat 0.85 in some paths  |
| 15    | Locale-aware date parsing                                               | US-style only today                |
| 18    | Ollama HA — retries + model fallback                                    | Single point of failure            |

None of these are deployment blockers; the system as committed handles uploads end-to-end with real risk scores.

---

## 11. Comparison vs. a .NET Legacy Architecture

The repo also contains a .NET reference implementation ([HPDDocValidator/](HPDDocValidator/)) built around C# + ASP.NET Core + PdfPig / OpenXml + a singleton in-memory `ReportStore` and a single "do everything" LLM prompt. It represents a typical legacy-style document-validation stack. The table below compares it to the Python (FastAPI + Ollama + PaddleOCR) stack we ship.

### Capability matrix

| Concern                                                            | .NET legacy (`HPDDocValidator`)                                           | Python stack (this repo)                                                            | Winner                                      |
| ------------------------------------------------------------------ | ------------------------------------------------------------------------- | ----------------------------------------------------------------------------------- | ------------------------------------------- |
| Cross-check user-claimed info vs document                          | **Not implemented**                                                       | `field_match.compare` per field (RapidFuzz / exact / fuzzy)                         | **Python**                                  |
| Per-applicant verdict across many docs                             | Per-file only                                                             | `rollupDocuments()` + worst-status precedence                                       | **Python**                                  |
| Persistence                                                        | **Singleton in-memory** `ReportStore` — lost on restart                   | Postgres + Prisma (durable, queryable, multi-instance safe)                         | **Python**                                  |
| Auth                                                               | None                                                                      | Bearer-token on every `/api/*` route                                                | **Python**                                  |
| Audit trail                                                        | Stdout logs only                                                          | Per-request JSON in `audit/YYYY-MM-DD/{run_id}.json` + request-id header            | **Python**                                  |
| Decision logic                                                     | Embedded in natural-language prompt — untestable                          | Deterministic `rules.py` + `score.py` — unit-tested                                 | **Python**                                  |
| Status vocabulary                                                  | 3 states (`Matched / Partial / Mismatch`)                                 | 6 states (`PASSED / FAILED / MANUAL_REVIEW / UNREADABLE / EXPIRED / DATA_MISMATCH`) | **Python**                                  |
| Image / scanned PDF support                                        | Delegated to a vision LLM (one big prompt)                                | Real OCR pipeline (PaddleOCR + `pypdfium2` text-layer fast-path)                    | **Python**                                  |
| Horizontal scale                                                   | Stateful singleton blocks multi-instance                                  | Stateless API after commit `27213a8` — N replicas                                   | **Python**                                  |
| Hot-path test coverage                                             | None shipped                                                              | 17 pytest tests (rules, field-match, HTTP, auth, validation)                        | **Python**                                  |
| Container story                                                    | `Dockerfile` only                                                         | Full `docker-compose` (UI + API + Ollama + healthchecks + secrets)                  | **Python**                                  |
| New document type                                                  | New C# class + recompile + redeploy                                       | Add prompt template + rule entry, no code change                                    | **Python**                                  |
| Hosting cost (5 K docs/mo)                                         | Windows VM + IIS or Linux + Kestrel ~ $80 / mo + Windows license overhead | $30–80 / mo Linux VM, no licence                                                    | **Python**                                  |
| LLM cost per request                                               | Same Ollama path                                                          | Same Ollama path                                                                    | **Tie**                                     |
| File-type extractor maturity                                       | PdfPig + OpenXml + EPPlus (excellent for text PDFs)                       | `pypdfium2` + `python-docx` + `openpyxl` + PaddleOCR                                | **Tie / slight Python edge** (covers scans) |
| Developer onboarding                                               | C# + .NET SDK + Visual Studio                                             | Python + `pip install` + uvicorn                                                    | **Python** (smaller barrier)                |
| Mature .NET tooling (profilers, BenchmarkDotNet, Roslyn analyzers) | Excellent                                                                 | Equivalent (pyinstrument, ruff, mypy) — different ecosystem                         | **.NET edge**                               |
| Strongly-typed compiled binary                                     | Yes                                                                       | Runtime-typed (Pydantic enforces at boundaries)                                     | **.NET edge**                               |

### Prompt reuse — what was ported, what was deliberately changed

The .NET work on the LLM prompt was the most valuable piece of the legacy app, so the per-type **field lists** and the **defensive JSON parser** were ported verbatim. The single monolithic prompt was **not** reused — it conflates five jobs the Python pipeline now handles separately.

| Aspect of the legacy prompt                                                                                                                                              | Status in Python                                                                                                                | Why                                                                                                                                 |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| Per-type field list (`GetFieldExtractionSpec`)                                                                                                                           | **Ported as-is** → [`FIELD_SPECS`](Web-api/app/pipeline/prompts.py)                                                             | The hard-won "which fields matter on a PayStub vs a BankStatement" knowledge is reused                                              |
| Strict-JSON instruction + example schema                                                                                                                                 | **Ported** (same "no markdown, no code fences" wording)                                                                         | Same model (`llama3.1:8b`), same failure modes                                                                                      |
| Defensive parser (strip ```fences, slice `{...}`)                                                                                                                        | **Ported** → [`llm.py`](Web-api/app/pipeline/llm.py) `_extract_json`                                                            | Llama still occasionally wraps output in prose                                                                                      |
| `BuildValidationPrompt` doing **classify + extract + validate + flag + recommend** in one call, with all 35 HPD rules inlined as prose ("Within 120 days of {today}", …) | **Deliberately replaced** with extraction-only prompts; rules moved to [`rules.py`](Web-api/app/pipeline/rules.py)              | Prose rules in a prompt are untestable, non-deterministic, drift between model versions, and can't be audited for compliance        |
| `validationStatus` / `recommendation` produced by the LLM                                                                                                                | **Deliberately removed**; decisions made by [`score.py`](Web-api/app/pipeline/score.py) on top of OCR confidence + rule outputs | A regulated workflow needs reproducible decisions — same inputs must produce the same status every time, which an LLM can't promise |
| Single big response object (`summary`, `flags`, `recommendation`, `reason`)                                                                                              | Split: `extracted_fields` from LLM, `issues` / `risk_score` / `status` from Python                                              | Clean separation of "what does the document say" (LLM) vs "what should we do about it" (code)                                       |

**Net effect:** the prompt the Python stack sends to Ollama is ~30 lines and asks for one thing — a JSON dict of the listed fields. Everything the legacy prompt asked the LLM to _decide_ now lives in deterministic, unit-tested Python.

### Where .NET would still be the right call

This is not a "Python always wins" pitch — .NET remains the right call when:

- The hosting organisation is already a 100 % .NET shop (operators, monitoring, identity).
- True CPU-bound parallelism _inside one process_ is required (real threads, no GIL).
- The target is Windows-only environments with strict signed-binary policies.

For **this** product (heterogeneous docs, cross-validation, LLM-in-the-loop, multi-tenant durability, horizontal scale, low-licence-cost target) the Python stack hits every requirement out of the box.

### Why the Python stack is better here, in one paragraph

The legacy .NET app treats the LLM as the entire decision-maker and keeps state in a process-local singleton. That makes it impossible to (a) compare a document to the user-claimed data, (b) aggregate a verdict across multiple documents, (c) survive a restart, (d) scale beyond a single instance, or (e) prove compliance with an audit log. The Python stack keeps the LLM constrained to **field extraction** and moves decision logic into deterministic, unit-tested Python — which is exactly what a regulated document-verification workflow needs.

---
