# TTB Label Verification

A proof-of-concept web application for checking alcohol beverage label images against structured application data.

The app accepts a label image plus seven application fields, extracts label text with a vision model, compares each field with a deliberate matching strategy, and returns per-field `PASS` / `FAIL` plus an overall `APPROVED` / `NEEDS_REVIEW` verdict.

## Live URLs

- Frontend: https://ttb-label-verification-frontend.vercel.app
- Backend health check: https://ttb-label-verification-backend.onrender.com/health

Date of last verification: July 31, 2026 (frontend loads, backend health returns `ok`, and `backend/scripts/live_smoke.py` returns a 200 verdict end-to-end).

> [!NOTE]
> The backend is hosted on Render's free tier, so the first request after inactivity may have a cold-start delay.

## Features

- Single-label verification flow.
- Batch verification flow with summary counts and per-label drill-down.
- Image preprocessing before model extraction to reduce latency and cost.
- Structured JSON extraction from OpenAI.
- Field-by-field comparison with expected-vs-found output.
- Stateless backend with no database or persisted uploads.
- Human-readable errors for invalid files, missing inputs, and provider failures.

## Architecture

```text
React/Vite frontend
  -> multipart/form-data image + JSON application data
FastAPI backend
  -> validation
  -> image preprocess
  -> OpenAI vision extraction
  -> comparison engine
  -> VerificationResult / BatchResult
```

Main backend endpoints:

- `GET /health`
- `POST /verify`
- `POST /verify/batch`

## Comparison Strategy

The comparison engine is pure Python logic and is tested without model calls.

| Field | Strategy |
| --- | --- |
| Brand Name | normalized fuzzy/contains match with minimum input length guard |
| Class / Type | normalized fuzzy/contains match, including whiskey/whisky handling |
| Alcohol Content | numeric percent comparison with `+/- 0.1` tolerance |
| Bottle Size | unit normalization to milliliters |
| Producer | normalized fuzzy/contains match |
| Country of Origin | normalized aliases such as `USA`, `U.S.A.`, `America`, `United States` |
| Government Warning | exact case-sensitive comparison after whitespace collapse only |

Any failed field returns `NEEDS_REVIEW`; all fields passing returns `APPROVED`.

## Tech Stack

- Backend: Python 3.12, FastAPI, Pydantic, Pillow, RapidFuzz, OpenAI Python SDK
- Vision model: `gpt-4.1-mini`, confirmed present in OpenAI's current model list (`client.models.list()`, checked July 13, 2026)
- Frontend: React, Vite, JavaScript, CSS
- Deployment: Render backend, Vercel frontend
- Dependency managers: `uv` for Python, `npm` for frontend

## Local Setup

Install backend dependencies:

```bash
cd backend
uv sync
```

Install frontend dependencies:

```bash
cd frontend
npm install
```

Create local environment files:

```bash
cp backend/.env.example backend/.env
cp frontend/.env.example frontend/.env
```

Set backend environment variables:

```text
CORS_ORIGINS=http://localhost:5173
OPENAI_API_KEY=<your OpenAI API key>
OPENAI_MODEL=gpt-4.1-mini
```

Set frontend environment variables:

```text
VITE_API_BASE_URL=http://localhost:8000
```

## Environment Variables

Every variable read by the code, across the backend (`backend/.env`), frontend (`frontend/.env`), and deployment platforms:

| Variable | Required | Default | Purpose |
| --- | --- | --- | --- |
| `OPENAI_API_KEY` | Yes | none | OpenAI API key used for vision extraction |
| `OPENAI_MODEL` | No | `gpt-4.1-mini` | OpenAI model used for label extraction |
| `CORS_ORIGINS` | No | `http://localhost:5173,http://127.0.0.1:5173` | Comma-separated origins allowed by backend CORS |
| `VISION_TIMEOUT_SECONDS` | No | `4.5` | Vision model request timeout in seconds |
| `MAX_IMAGE_DIMENSION` | No | `1024` | Longest image side in pixels after preprocessing downscale |
| `JPEG_QUALITY` | No | `75` | JPEG quality used when re-encoding preprocessed images |
| `MAX_IMAGE_BYTES` | No | `10485760` | Maximum accepted upload size per image, in bytes |
| `BATCH_CONCURRENCY` | No | `3` | Maximum concurrent vision requests while processing a batch |
| `MAX_BATCH_SIZE` | No | `10` | Maximum labels accepted per `/verify/batch` request |
| `VITE_API_BASE_URL` | No | `http://localhost:8000` | Backend base URL the frontend calls |
| `PYTHON_VERSION` | Render only | `3.12.13` | Pins the Python runtime version on Render |
| `PYTHONUNBUFFERED` | Render only | `1` | Emits backend logs immediately instead of buffering |

## Run Locally

Start the backend:

```bash
cd backend
uv run uvicorn app.main:app --host 0.0.0.0 --port 8000
```

Start the frontend in another terminal:

```bash
cd frontend
npm run dev
```

Local URLs:

- Backend health: http://localhost:8000/health
- Frontend: http://localhost:5173

## API Examples

The examples below hit the deployed backend; swap the base URL for `http://localhost:8000` to hit a local run.

### POST /verify

```bash
curl -X POST https://ttb-label-verification-backend.onrender.com/verify \
  -F "image=@label.jpg" \
  -F 'application_data={
    "brand_name": "Old Tom Distillery",
    "class_type": "Straight Bourbon Whiskey",
    "abv": "45%",
    "net_contents": "750 mL",
    "producer": "Old Tom Spirits Co.",
    "country_of_origin": "United States",
    "government_warning": "GOVERNMENT WARNING: (1) ACCORDING TO THE SURGEON GENERAL, WOMEN SHOULD NOT DRINK ALCOHOLIC BEVERAGES DURING PREGNANCY BECAUSE OF THE RISK OF BIRTH DEFECTS. (2) CONSUMPTION OF ALCOHOLIC BEVERAGES IMPAIRS YOUR ABILITY TO DRIVE A CAR OR OPERATE MACHINERY, AND MAY CAUSE HEALTH PROBLEMS."
  }'
```

Success response (`200`), one entry per field:

```json
{
  "results": [
    {
      "field": "brand_name",
      "match_type": "fuzzy_contains",
      "expected": "Old Tom Distillery",
      "found": "OLD TOM DISTILLERY",
      "status": "PASS"
    },
    {
      "field": "abv",
      "match_type": "numeric_tolerance",
      "expected": "45%",
      "found": "45% ALC./VOL. (90 PROOF)",
      "status": "PASS"
    }
  ],
  "overall_verdict": "APPROVED",
  "latency_ms": 2431
}
```

`overall_verdict` is `APPROVED` when every field passes, otherwise `NEEDS_REVIEW`. Error responses use a single `detail` string:

```json
{"detail": "Request must include image and application_data."}
{"detail": "Unsupported image type. Use JPEG, PNG, or WebP."}
```

### POST /verify/batch

Repeat `images` once per file; `application_data` is a JSON array in the same order:

```bash
curl -X POST https://ttb-label-verification-backend.onrender.com/verify/batch \
  -F "images=@bourbon.jpg" \
  -F "images=@gin.jpg" \
  -F 'application_data=[
    {"brand_name": "Old Tom Distillery", "class_type": "Straight Bourbon Whiskey", "abv": "45%", "net_contents": "750 mL", "producer": "Old Tom Spirits Co.", "country_of_origin": "United States", "government_warning": "GOVERNMENT WARNING: (1) ACCORDING TO THE SURGEON GENERAL, WOMEN SHOULD NOT DRINK ALCOHOLIC BEVERAGES DURING PREGNANCY BECAUSE OF THE RISK OF BIRTH DEFECTS. (2) CONSUMPTION OF ALCOHOLIC BEVERAGES IMPAIRS YOUR ABILITY TO DRIVE A CAR OR OPERATE MACHINERY, AND MAY CAUSE HEALTH PROBLEMS."},
    {"brand_name": "Juniper Ridge", "class_type": "Dry Gin", "abv": "40%", "net_contents": "700 mL", "producer": "Juniper Ridge Distillers", "country_of_origin": "England", "government_warning": "GOVERNMENT WARNING: (1) ACCORDING TO THE SURGEON GENERAL, WOMEN SHOULD NOT DRINK ALCOHOLIC BEVERAGES DURING PREGNANCY BECAUSE OF THE RISK OF BIRTH DEFECTS. (2) CONSUMPTION OF ALCOHOLIC BEVERAGES IMPAIRS YOUR ABILITY TO DRIVE A CAR OR OPERATE MACHINERY, AND MAY CAUSE HEALTH PROBLEMS."}
  ]'
```

Success response (`200`); items that fail individually (for example an unsupported file type) come back as `FAILED` entries with an `error` message instead of failing the whole batch:

```json
{
  "items": [
    {
      "index": 0,
      "filename": "bourbon.jpg",
      "status": "COMPLETED",
      "verification": {
        "results": ["... same shape as the single /verify response ..."],
        "overall_verdict": "APPROVED",
        "latency_ms": 2431
      },
      "error": null
    },
    {
      "index": 1,
      "filename": "gin.jpg",
      "status": "FAILED",
      "verification": null,
      "error": "Unsupported image type. Use JPEG, PNG, or WebP."
    }
  ],
  "summary": {"passed": 1, "needs_review": 1, "total": 2}
}
```

Whole-batch errors still use `detail`, for example when the batch exceeds `MAX_BATCH_SIZE`:

```json
{"detail": "Batch is too large. Maximum is 10 labels per request."}
```

## Tests And Build

Run backend tests:

```bash
cd backend
uv run pytest
```

If `uv` is not on your shell path but the virtual environment already exists:

```bash
cd backend
./.venv/bin/pytest
```

Run frontend build:

```bash
cd frontend
npm run build
```

## Deployment

### Render Backend

Create a Render Web Service connected to the repo.

Use these settings:

- Runtime: Python
- Root directory: `backend`
- Build command: `pip install uv && uv sync --frozen --no-dev`
- Start command: `.venv/bin/uvicorn app.main:app --host 0.0.0.0 --port $PORT`
- Health check path: `/health`
- Plan: Free

Required Render environment variables:

```text
PYTHON_VERSION=3.12.13
PYTHONUNBUFFERED=1
CORS_ORIGINS=https://ttb-label-verification-frontend.vercel.app
OPENAI_API_KEY=<set in Render dashboard>
OPENAI_MODEL=gpt-4.1-mini
```

### Vercel Frontend

Create a Vercel project connected to the repo.

Use these settings:

- Framework preset: Vite
- Root directory: repository root
- Install command: `cd frontend && npm ci`
- Build command: `cd frontend && npm run build`
- Output directory: `frontend/dist`

Required Vercel environment variable:

```text
VITE_API_BASE_URL=https://ttb-label-verification-backend.onrender.com
```

## Live Smoke Check

Exercise the deployed backend end-to-end (image upload, vision extraction, comparison, JSON response):

```bash
cd backend
uv run python scripts/live_smoke.py
```

The script generates a synthetic sample label, posts it with matching application data to `POST /verify` on the deployed URL, and exits non-zero unless the response is 2xx. It prints the overall verdict and server latency on success. No local image is required; use `--image path/to/label.jpg` to smoke-test with a real label, or `--url http://localhost:8000` to point it at a local backend. The default timeout is generous (90s) because the first request after Render free-tier inactivity is a cold start.

## Performance

Measured single-label `POST /verify` latency (July 13, 2026, current configuration):

| Percentile | Client-observed total | Server-reported `latency_ms` |
| --- | --- | --- |
| p50 | 3327 ms | 3325 ms |
| p95 | 4272 ms | 4270 ms |

How the measurement was taken: `backend/scripts/measure_latency.py` posted a synthetic sample label JPEG to `/verify` sequentially — 15 timed runs after 1 untimed warmup request, 0 failures — and reports nearest-rank p50/p95 of the client-observed request time. The server-reported `latency_ms` (which excludes network transfer) is recorded alongside it; the vision model call dominates it, with preprocessing and comparison contributing only a few milliseconds. The backend ran locally with the deployed configuration (`gpt-4.1-mini`, 1024 px / quality-75 preprocessing, 4.5 s timeout). Reproduce with:

```bash
cd backend
uv run python scripts/measure_latency.py path/to/label1.jpg path/to/label2.jpg
```

Latency notes:

- Output length dominates vision latency. The extraction prompt instructs the model not to transcribe the full label into `raw_text` (nothing consumes it); with transcription on, the same measurement showed roughly double the median extraction time on the same label with otherwise identical field output.
- The vision model request timeout is 4.5 seconds (`VISION_TIMEOUT_SECONDS`), passed directly to the OpenAI SDK as the per-request timeout. The client is built with automatic retries disabled so the timeout is a real per-request budget rather than a floor.
- The Render free tier cold-starts after inactivity, which adds up to ~30 seconds to the first request only; warmup requests absorb this and are excluded from the numbers above.

## Approach And Tools

The project was built AI-first with a Plan/Review/Execute loop, with a human reviewing and committing every change:

- The initial application (FastAPI backend, comparison engine, React frontend, deployment setup) was built with OpenAI Codex: each piece of work started as a written plan, the generated changes were reviewed before execution, and only reviewed diffs were committed.
- The follow-up hardening passes (correctness fixes, UX/quality, frontend tests + CI, deployment/documentation, and a vision provider swap from Gemini to OpenAI) were implemented with OpenAI Codex working from direct instructions: the agent implemented and verified each requested change (running pytest, Vitest, builds, and live API checks), and every diff was reviewed, exercised in the running app, and staged/committed by hand.
- AI-generated vs hand-written: the bulk of the application code, the test suites, the CI workflow, and this README were AI-generated under human review. Branch and PR management, all git operations, deployment configuration in the Render and Vercel dashboards, latency measurement runs against the deployed URL, and final acceptance testing in the browser were done by hand.
- Decisions where the model was overridden: the AI's first pass at the batch cap allowed 20 labels and only enforced it server-side — it was lowered to 10 with client-side blocking of additional cards, because 20 large uploads risked oversized requests. The AI's disabled-button styling showed a busy cursor on cap-disabled buttons — it was changed to a not-allowed cursor so "blocked" and "busy" read differently. The AI was also barred from running git staging/commit/push commands; all version control was kept human-only.

## Assumptions

- The application data entered in the form is the authoritative reference; the label image is what gets checked against it.
- Each uploaded image shows a single label that is reasonably legible (not heavily blurred, angled, or glare-obscured).
- The standard TTB government warning (27 CFR 16.21) is legally mandatory, so a label without a readable warning should never pass.
- Verdicts support a human review workflow; `NEEDS_REVIEW` means "a person should look", not "rejected".

## Limitations

- The app is stateless; it does not save uploads, results, or user sessions, so there is no history or audit trail.
- Vision extraction can vary with image quality, glare, angle, blur, and provider availability.
- OpenAI provider latency or rate limiting can cause occasional slow or failed extraction requests.
- Render free tier may cold start after inactivity, delaying the first request by up to ~30 seconds.
- The proof-of-concept is intended for review workflow support, not automatic legal approval.

## Tradeoffs

- Government warning matching is intentionally strict and case-sensitive; OCR mistakes produce `NEEDS_REVIEW` so a human inspects the extracted text, at the cost of more false alarms.
- Full-label `raw_text` transcription is disabled in the extraction prompt for a roughly 2x latency win; the seven compared fields and the verbatim government warning were identical in A/B testing, and the `raw_text` schema field is kept (returned as null) so transcription can be re-enabled by prompt change alone.
- Images are downscaled to 1024 px and re-encoded at JPEG quality 75 before extraction to cut latency and cost, trading away fine detail that could matter for very small print.
- Fuzzy/contains matching on brand, class, and producer tolerates OCR noise and partial label phrases, at the cost of occasionally passing a near-miss; a minimum-length guard limits false positives on short names.
- Statelessness keeps the deployment simple and avoids storing user data, at the cost of persistence features like saved batches or result history.
