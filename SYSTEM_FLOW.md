# Post-Video-Classifier: Complete System Flow

## Table of Contents

1. [System Overview](#system-overview)
2. [Architecture Diagram](#architecture-diagram)
3. [Application Bootstrap](#application-bootstrap)
4. [API Entry Points](#api-entry-points)
5. [Core Processing Pipeline: `/classify`](#core-processing-pipeline-classify)
6. [Query Pipeline: `/query` and `/query/two-stage`](#query-pipeline)
7. [Batch Processing Pipelines](#batch-processing-pipelines)
8. [Health & Monitoring Endpoints](#health--monitoring-endpoints)
9. [Service Layer Deep Dive](#service-layer-deep-dive)
10. [Data Models & Classification Schema](#data-models--classification-schema)
11. [Deduplication System](#deduplication-system)
12. [Error Handling & Retry Logic](#error-handling--retry-logic)
13. [Configuration & Environment](#configuration--environment)

---

## System Overview

The post-video-classifier is a Flask-based REST API that classifies TikTok influencer videos using Google's Gemini model (via Vertex AI) with DSPy prompt optimization. It extracts video frames, analyzes content through multimodal AI, and stores classification embeddings in a Jina vector database for semantic search.

**Core responsibilities:**
- Accept video URLs (public, signed, or `gs://` URIs)
- Analyze video content using Gemini 2.5 Flash with DSPy-optimized prompts
- Extract and upload representative frames to Google Cloud Storage
- Store text embeddings and metadata in Jina for vector search
- Deduplicate videos using composite IDs with native Jina upsert
- Support semantic search over classified videos

---

## Architecture Diagram

```
                         ┌─────────────────────────────────────────────┐
                         │              Flask Application              │
                         │  (CORS, Request Logging, Error Handlers)    │
                         └──────────────────┬──────────────────────────┘
                                            │
              ┌─────────────────────────────┼─────────────────────────────┐
              │                             │                             │
    ┌─────────▼────────┐       ┌────────────▼──────────┐     ┌───────────▼──────────┐
    │  classify_bp     │       │     query_bp          │     │   sequential_bp      │
    │  POST /classify  │       │  POST /query          │     │   bucket_bp          │
    │                  │       │  POST /query/two-stage │     │   batch_classify_bp  │
    └────────┬─────────┘       └────────────┬──────────┘     └───────────┬──────────┘
             │                              │                            │
    ┌────────▼─────────────────────────────────────────────────────────────────────┐
    │                           Service Layer (Lazy-Loaded Singletons)             │
    │                                                                              │
    │  ┌──────────────────┐  ┌────────────────┐  ┌──────────────┐                  │
    │  │ DSPyGeminiService│  │ StorageService │  │ QueryService │                  │
    │  │ (AI Analysis)    │  │ (Vector Store) │  │ (Search)     │                  │
    │  └────────┬─────────┘  └───────┬────────┘  └──────┬───────┘                  │
    │           │                    │                   │                          │
    │  ┌────────▼─────────┐  ┌──────▼────────┐          │                          │
    │  │ VideoProcessor   │  │  JinaClient   │◄─────────┘                          │
    │  │ (Frame Extract)  │  │  (API Client) │                                     │
    │  └────────┬─────────┘  └───────────────┘                                     │
    │           │                                                                  │
    │  ┌────────▼─────────┐  ┌───────────────┐  ┌──────────────┐                   │
    │  │   GCSService     │  │ LangSmith     │  │ MetricsService│                  │
    │  │ (Cloud Storage)  │  │ Service       │  │ (Performance) │                  │
    │  └──────────────────┘  └───────────────┘  └──────────────┘                   │
    └──────────────────────────────────────────────────────────────────────────────┘
                         │                    │                    │
              ┌──────────▼──────┐  ┌──────────▼──────┐  ┌────────▼──────────┐
              │   Vertex AI     │  │   Jina API      │  │  Google Cloud     │
              │ (Gemini 2.5     │  │  (Vector DB)    │  │  Storage (GCS)    │
              │  Flash)         │  │                 │  │                   │
              └─────────────────┘  └─────────────────┘  └───────────────────┘
```

---

## Application Bootstrap

**Entry point:** `src/api/app.py` → `create_app()`

The Flask application is assembled through a factory pattern:

1. **Environment loading** — Reads `.env` from `simple/.env` via `python-dotenv`
2. **Container configuration** — Validates environment via `ContainerConfig.from_environment()`
3. **CORS** — Enabled with configurable origins (`CORS_ORIGINS` env var, defaults to `*`)
4. **Blueprint registration** — Each functional area is a Flask Blueprint:

   | Blueprint | URL Prefix | Purpose |
   |-----------|------------|---------|
   | `classify_bp` | `/` | Single video classification |
   | `query_bp` | `/` | Vector search |
   | `health_bp` | `/` | Basic health check |
   | `enhanced_health_bp` | `/` | Kubernetes probes (`/health/ready`, `/health/live`) |
   | `metrics_bp` | `/` | Performance metrics |
   | `bucket_bp` | `/bucket` | GCS bucket processing |
   | `sequential_bp` | `/api/v1/sequential` | Sequential batch ingestion |
   | `batch_classify_bp` | `/` | Gemini Batch API (50% cost savings) |
   | `batch_ingest_bp` | `/` | Batch ingestion endpoints |
   | `docs_bp` | `/` | API documentation |

5. **Deprecated route handling** — `POST /batch/classify` returns `410 Gone` with migration guide headers
6. **Error handlers** — Registered for 400, 404, 413, 500, and custom `VideoClassifierException`
7. **Request middleware** — Logs incoming requests, attaches `X-Request-ID` header to responses
8. **Deprecation middleware** — `DeprecationHandler` adds deprecation warnings where applicable

**Service initialization** is lazy — services are created on first use via getter functions (`get_gemini_service()`, `get_storage_service()`, etc.) stored as module-level singletons. This avoids startup failures when external services are unreachable and reduces memory usage.

---

## API Entry Points

### Complete Endpoint Map

| Method | Path | Status | Purpose |
|--------|------|--------|---------|
| `POST` | `/classify` | Active | Single video classification (primary endpoint) |
| `POST` | `/batch/classify` | **Deprecated (410)** | Replaced by `/api/v1/sequential/process` |
| `POST` | `/query` | Active | Basic semantic vector search |
| `POST` | `/query/two-stage` | Active | Two-stage retrieval (dense + rerank) |
| `GET` | `/health` | Active | Full health check with dependency status |
| `GET` | `/health/ready` | Active | Kubernetes readiness probe |
| `GET` | `/health/live` | Active | Kubernetes liveness probe |
| `GET` | `/metrics` | Active | Performance metrics |
| `GET` | `/metrics/ingestion` | Active | Ingestion-specific metrics |
| `GET` | `/metrics/memory` | Active | Memory usage metrics |
| `POST` | `/metrics/memory/baseline` | Active | Set memory baseline |
| `GET` | `/metrics/resources` | Active (dev only) | Active resource list |
| `POST` | `/bucket/process` | Active | Process videos from GCS bucket |
| `GET` | `/bucket/status/{job_id}` | Active | Bucket job status |
| `POST` | `/api/v1/sequential/process` | Active | Start sequential batch processing |
| `GET` | `/api/v1/sequential/status/{session_id}` | Active | Session status |
| `POST` | `/api/v1/sequential/resume/{session_id}` | Active | Resume paused session |
| `GET` | `/api/v1/sequential/items/{session_id}` | Active | Get processed items |
| `POST` | `/classify-batch` | Active | Submit Gemini Batch API job (50% cost savings) |
| `GET` | `/classify-batch/jobs` | Active | List batch jobs |
| `GET` | `/classify-batch/status/{job_id}` | Active | Batch job status |
| `POST` | `/classify-batch/results/{job_id}` | Active | Process batch results into Jina |

---

## Core Processing Pipeline: `/classify`

This is the primary endpoint. The flow differs based on request format (v2.0 enhanced vs v1.0 legacy).

### v2.0 Enhanced Flow (signedUrl format)

This is the current production path. Triggered when the request body contains `signedUrl`.

```
Client Request                    Response
     │                               ▲
     ▼                               │
┌────────────────────────────────────────────────────────────────┐
│ 1. REQUEST VALIDATION                                          │
│    Parse JSON body                                             │
│    Validate via EnhancedClassifyRequest (Pydantic model)       │
│    Required: signedUrl, mediaId                                │
│    Optional: userHandle, soundId                               │
└────────────────────┬───────────────────────────────────────────┘
                     │
                     ▼
┌────────────────────────────────────────────────────────────────┐
│ 2. GEMINI ANALYSIS (DSPyGeminiService.analyze_video)           │
│                                                                │
│    a. Upload video to Gemini Files API (temporary)             │
│    b. Build DSPy-optimized prompt with classification schema   │
│    c. Send video + prompt to Gemini 2.5 Flash via Vertex AI    │
│    d. Parse structured JSON response                           │
│    e. Validate against schema (categories, tags, genres, etc.) │
│    f. Normalize legacy music genre names                       │
│    g. Extract first frame → upload to GCS                      │
│    h. Clean up Gemini temporary file                           │
│                                                                │
│    Output: Dict with ~17 classification fields + frame_url     │
└────────────────────┬───────────────────────────────────────────┘
                     │
                     ▼
┌────────────────────────────────────────────────────────────────┐
│ 3. METADATA ASSEMBLY                                           │
│                                                                │
│    Extract from Gemini response:                               │
│    - categories (1-3 from 31 options)                          │
│    - tags (0-3 from 8 style tags)                              │
│    - language (1 from 16 options)                              │
│    - common_music (0-3 from 25 genres)                         │
│    - unrestricted_genre (1-3 creative genre tags)              │
│    - background_music_mode (mood/tone descriptors)             │
│    - background_music_audibility (1-5 scale)                   │
│    - music_proportion (0.000-1.000)                            │
│    - account_type (Human/Pet/Non-Human)                        │
│    - content_purpose (Reviewer/Entertainer/Educator/Other)     │
│    - music_record_campaign_recommendation (Yes/No/Cannot Decide)│
│    - confidence_scores (per-category float scores)             │
│    - people (visual description, 0-500 chars)                  │
│    - video_description (800-2500 chars, music metadata inline) │
│                                                                │
│    Generate:                                                   │
│    - tiktok_url: https://tiktok.com/{userHandle}/video/{mediaId}│
│    - pulse_video_id: {userHandle}:{mediaId} (composite key)    │
│    - created_at / updated_at timestamps                        │
│    - v: 3 (schema version)                                     │
│    - ingestion_count: 1                                        │
└────────────────────┬───────────────────────────────────────────┘
                     │
                     ▼
┌────────────────────────────────────────────────────────────────┐
│ 4. TEXT CONTENT PREPARATION                                    │
│                                                                │
│    Primary: video_description from Gemini (800-2500 chars)     │
│    Fallback: Auto-generated from categories + tags + handle    │
│    (Jina requires non-empty text for embedding generation)     │
│                                                                │
│    The video_description contains embedded music metadata      │
│    as natural language sentences, making music characteristics  │
│    searchable via vector similarity.                           │
└────────────────────┬───────────────────────────────────────────┘
                     │
                     ▼
┌────────────────────────────────────────────────────────────────┐
│ 5. JINA VECTOR INGESTION (StorageService → JinaClient)         │
│                                                                │
│    Path A: Multimodal (if frame_storage_path exists)            │
│      StorageService.ingest(text, metadata, image_url)          │
│      → JinaClient.ingest_multimodal()                          │
│      → POST {JINA_API}/api/v1/ingest/multimodal               │
│                                                                │
│    Path B: Text-only (no frame available)                       │
│      StorageService.ingest(text, metadata)                     │
│      → StorageService.store_video_classification()             │
│      → JinaClient.ingest_text_only()                           │
│      → POST {JINA_API}/api/v1/ingest/text                     │
│                                                                │
│    Both paths use:                                             │
│    - id_field: "pulse_video_id" (enables native upsert)        │
│    - collection: pulse-videos-v2                               │
│    - Retry with exponential backoff (3 attempts, 1s/2s/4s)     │
│                                                                │
│    Deduplication: If pulse_video_id already exists, Jina       │
│    performs an upsert (update) instead of insert (create).     │
└────────────────────┬───────────────────────────────────────────┘
                     │
                     ▼
┌────────────────────────────────────────────────────────────────┐
│ 6. METRICS RECORDING                                           │
│                                                                │
│    MetricsService.record_ingestion()                           │
│    Tracks: method, success/failure, processing time, doc ID    │
└────────────────────┬───────────────────────────────────────────┘
                     │
                     ▼
┌────────────────────────────────────────────────────────────────┐
│ 7. RESPONSE CONSTRUCTION                                       │
│                                                                │
│    Success (200):                                              │
│    {                                                           │
│      "success": true,                                          │
│      "operation": "create" | "update",                         │
│      "pulse_video_id": "@handle:mediaId",                      │
│      "data": {                                                 │
│        "category": [...],                                      │
│        "tags": [...],                                          │
│        "video_description": "...",                              │
│        "language": "English",                                  │
│        "common_music": [...],                                  │
│        "frame_url": "gs://...",                                │
│        "vector_id": "...",                                     │
│        "processing_time_ms": 2345.67,                          │
│        ... (all classification fields)                         │
│      },                                                        │
│      "processing_time_ms": 2345.67,                            │
│      "ingestion_count": 1                                      │
│    }                                                           │
│                                                                │
│    Error (400/500):                                            │
│    EnhancedErrorResponse with error code and details           │
└────────────────────────────────────────────────────────────────┘
```

### v1.0 Legacy Flow (video/post_id format)

Triggered when the request body contains `video` instead of `signedUrl`. Supports local file uploads in addition to URLs.

Key differences from v2.0:
- Uses `ClassifyRequest` model (video, video_type, post_id, handle, timestamp)
- For local files: extracts frame via `VideoProcessor`, uploads to GCS separately
- For images: uses `analyze_frame()` instead of `analyze_video()`
- Creates `VideoClassification` dataclass instead of Pydantic model
- Storage path may use `store_classification_with_url()` with signed URLs

---

## Query Pipeline

### `POST /query` — Basic Semantic Search

```
Request                              Response
{                                    {
  "query": "fashion videos",           "results": [...],
  "limit": 10,                         "total": 42,
  "offset": 0,                         "query_time_ms": 123.4,
  "metadata_filters": {...},            "query": "fashion videos",
  "score_threshold": 0.5               "limit": 10,
}                                       "offset": 0
                                     }
```

**Flow:**
1. Validate `QueryRequest` (query length, limit 1-100, score threshold 0.0-1.0)
2. `QueryService.search()` → Sends query to Jina for dense vector search
3. Applies metadata filters and score threshold
4. Returns ranked results with metadata

### `POST /query/two-stage` — Enhanced Retrieval

Two-stage retrieval improves search quality by first casting a wide net, then reranking:

```
Stage 1: Dense Search                    Stage 2: Reranking
┌──────────────────────┐                ┌──────────────────────┐
│ Retrieve 50 candidates│ ──────────▶   │ Rerank top 10        │
│ (initial_pool_size)   │               │ (rerank_top_n)       │
│ via vector similarity │               │ via cross-encoder    │
└──────────────────────┘                └──────────┬───────────┘
                                                   │
                                                   ▼
                                        Return final results
                                        (limit, default: 10)
```

**Additional request fields:**
- `initial_pool_size` (default: 50) — Stage 1 candidate count
- `rerank_top_n` (default: 10) — Stage 2 reranking count

**Response includes:**
- `retrieval_method`: "two_stage"
- `stages`: Performance details per stage

---

## Batch Processing Pipelines

### Sequential Ingestion (`/api/v1/sequential/...`)

For processing multiple videos from a GCS bucket with rate limiting and resume capability.

```
POST /api/v1/sequential/process
{
  "bucket_name": "pulse_video_frames",
  "prefix": "videos/",
  "max_items": 100,
  "rate_limit": 10,        ← requests/second (1-100)
  "max_retries": 3,        ← per-video retry count (0-10)
  "continue_on_error": true
}
→ 202 Accepted { "session_id": "uuid", ... }
```

**Processing flow:**
1. `SequentialIngestionService.start_processing()` scans the GCS bucket
2. Lists objects matching the prefix, up to `max_items`
3. Processes each video sequentially with rate limiting (token bucket)
4. For each video: downloads → classifies via Gemini → stores in Jina
5. Tracks progress per session (pending → processing → completed/failed)

**Session management:**
- `GET /api/v1/sequential/status/{session_id}` — Progress, counts, ETA
- `POST /api/v1/sequential/resume/{session_id}` — Resume paused/failed session
- `GET /api/v1/sequential/items/{session_id}` — List processed items (filterable by status)

**Session states:** `pending` → `processing` → `completed` | `failed` | `paused`

### Bucket Processing (`/bucket/...`)

Simpler bucket processing with timestamp-based filtering:

```
POST /bucket/process
{
  "timestamp": "2024-01-01T00:00:00Z",  ← only process videos after this time
  "bucket": "pulse_videos",
  "prefix": "videos/",
  "max_videos": 100,
  "force_reprocess": false
}
```

Uses `BucketScannerService` to scan GCS, filter by modification timestamp, and process in bulk.

### Gemini Batch API (`/classify-batch/...`)

Uses Vertex AI's Batch Prediction API for **50% cost savings** on bulk classification:

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│ 1. Submit    │───▶│ 2. Poll      │───▶│ 3. Process   │───▶│ 4. Ingest    │
│ POST         │    │ GET status/  │    │ POST results/│    │ to Jina      │
│ /classify-   │    │ {job_id}     │    │ {job_id}     │    │              │
│ batch        │    │              │    │              │    │              │
│              │    │ Wait for     │    │ Parse Gemini │    │ Store each   │
│ Creates JSONL│    │ "succeeded"  │    │ output JSONL │    │ classification│
│ on GCS       │    │ status       │    │ from GCS     │    │ as vector    │
└──────────────┘    └──────────────┘    └──────────────┘    └──────────────┘
```

**GCS storage structure:**
```
gs://pulse_batch_jobs/
├── inputs/{job_id}.jsonl        # Batch input file
├── jobs/{job_id}/
│   ├── state.json               # Job metadata
│   └── items.json               # Video metadata for correlation
└── outputs/{job_id}/
    └── predictions.jsonl        # Gemini classification results
```

---

## Health & Monitoring Endpoints

### `GET /health` — Full Health Check

Checks all three external dependencies and returns aggregate status:

```
{
  "status": "healthy" | "unhealthy",
  "timestamp": "2024-02-12T10:30:00Z",
  "dependencies": {
    "gemini_api": "healthy" | "unhealthy" | "error",
    "jina_api":   "healthy" | "unhealthy" | "error",
    "gcs":        "healthy" | "unhealthy" | "error"
  },
  "container_id": "abc123",
  "version": "0.1.0",
  "uptime_seconds": 3600,
  "deprecation_warnings": []
}
```

- Returns **200** if all dependencies healthy, **503** if any unhealthy
- `container_id`, `version`, `uptime_seconds` from `get_container_info()`

### `GET /health/ready` — Kubernetes Readiness

Returns 200 if critical services (Gemini, Jina) are available. Used by Kubernetes to determine if the pod should receive traffic.

### `GET /health/live` — Kubernetes Liveness

Always returns 200 if the process is running. Used by Kubernetes to determine if the pod needs restart.

### `GET /metrics` — Performance Metrics

Returns operation counts, timings, and error rates from the in-memory metrics tracker.

### `GET /metrics/ingestion` — Ingestion Metrics

Compares URL-based vs file-based ingestion performance. Supports `?period=last_hour|last_day|last_week`.

### `GET /metrics/memory` — Memory Metrics

Returns current RSS/VMS memory usage, baseline comparison, growth rate, and active resource count via `psutil`.

---

## Service Layer Deep Dive

### DSPyGeminiService (`src/services/dspy_gemini_service.py`)

The AI analysis engine. Uses DSPy for optimized prompting against Gemini.

**Initialization:**
- Configures Vertex AI client (`google.genai`) with project credentials
- Loads DSPy with Gemini as the language model
- Optionally loads optimized DSPy model from `src/services/ai-models/`

**`analyze_video(url, timestamp, media_id, user_handle, sound_id)` flow:**
1. Upload video to Gemini Files API (creates temporary file reference)
2. Build the classification prompt using `VideoClassificationSignature` — includes the full schema of 31 categories, 8 style tags, 16 languages, 25 music genres with descriptions and examples
3. Call Gemini via `genai.Client.models.generate_content()` with `response_mime_type="application/json"`
4. Parse the structured JSON response
5. Validate each field against the schema using `ClassificationValidator`
6. Normalize legacy music genre names (e.g., map old names to current taxonomy)
7. Extract first frame via OpenCV, upload to GCS, attach `frame_url` and `frame_storage_path`
8. Clean up the temporary Gemini file (with retry on failure)
9. Return the complete classification dict

**Key DSPy fields in `VideoClassificationSignature`:**
- `reasoning` — Chain-of-thought analysis
- `video_description` — 800-2500 char description with embedded music metadata
- `people` — Visual characteristics (0-500 chars)
- `account_type` — Human / Pet / Non-Human
- `content_purpose` — Reviewer / Entertainer / Educator / Other
- `categories` — 1-3 from 31 content categories
- `style_tags` — 0-3 from 8 style tags
- `language` — 1 from 16 languages
- `music_genres` — 0-3 from 25 genres
- `unrestricted_genre` — 1-3 creative genre tags (parallel)
- `background_music_mode` — Mood/tone descriptors
- `background_music_audibility` — 1-5 scale
- `music_proportion` — 0.000-1.000
- `music_record_campaign_recommendation` — Yes / No / Cannot Decide
- `confidence_scores` — Per-category confidence
- `reviewer_specificity` — Conditional field for Reviewer content_purpose

### StorageService (`src/services/storage_service.py`)

Thin wrapper over `JinaClient` that handles:
- Text content validation (non-empty required)
- Image URL format validation (`http://`, `https://`, or `gs://`)
- Composite ID generation (`pulse_video_id`) if not already present
- Ingestion count tracking
- Backward-compatible `ingest()` method for the v2.0 API route

### JinaClient (`src/services/jina_client.py`)

Low-level HTTP client for the Jina vector database API.

**Connection:**
- Base URL: `{JINA_API_URL}:{JINA_API_PORT}/api/v1` (default: `http://34.132.139.92:5000/api/v1`)
- Collection: `pulse-videos-v2` (configurable via `COLLECTION_NAME`)
- Uses `requests.Session` for connection pooling
- Optional API key authentication via Bearer token

**Ingestion methods:**
- `ingest_multimodal(text, image_url, metadata)` → `POST /api/v1/ingest/multimodal`
- `ingest_text_only(text, metadata)` → `POST /api/v1/ingest/text`

Both methods include `id_field: "pulse_video_id"` in the payload to enable native Jina upsert.

**Retry logic (`_execute_with_retry`):**
- 3 attempts maximum
- Exponential backoff: 1s → 2s → 4s
- Retryable: 5xx errors, 429 rate limit, timeouts, connection errors
- Non-retryable: 4xx errors (except 429), validation errors

### QueryService (`src/services/query_service.py`)

**`search(query, limit, offset, metadata_filters, score_threshold)`:**
- Sends query text to Jina for dense vector search
- Applies metadata filters (eq, in operators)
- Filters results by score threshold
- Returns ranked results with metadata

**`two_stage_retrieval(query, limit, metadata_filters, initial_pool_size, rerank_top_n)`:**
1. Stage 1: Dense search retrieves `initial_pool_size` candidates (default 50)
2. Stage 2: Reranks top `rerank_top_n` candidates (default 10) using cross-encoder
3. Returns final `limit` results

### GCSService (`src/services/gcs_service.py`)

Manages frame image lifecycle in Google Cloud Storage.

- **Bucket:** `pulse_video_frames` (configurable via `GCS_BUCKET_NAME`)
- **Upload:** `upload_frame(frame_data, post_id)` — Uploads optimized JPEG frame
- **Signed URLs:** Generates v4 signed URLs with configurable expiration (default 3600s)
- **Auth:** Supports service account JSON and Application Default Credentials (ADC)

### VideoProcessor (`src/services/video_processor.py`)

OpenCV-based video frame extraction.

- `extract_first_frame(video_path)` — Reads first frame from video
- `extract_frame_at_timestamp(video_path, timestamp)` — Seeks to specific time
- `optimize_frame_for_upload(frame_path)` — Compresses JPEG for GCS
- `get_video_duration(video_path)` — Returns total duration
- Supports: .mp4, .avi, .mov, .mkv, .webm, .flv
- Handles both local files and HTTP URLs (via temporary download)

### LangSmithService (`src/services/langsmith_service.py`)

Optional observability layer for tracking LLM interactions.

- Records prompts, responses, latency, and token usage
- Filters sensitive data (video content, image data) before sending
- Enabled via `LANGSMITH_TRACING=true` environment variable
- Project: configurable via `LANGSMITH_PROJECT` (default: `pulse-rag`)

### MetricsService (`src/services/metrics_service.py`)

In-memory performance tracking.

- Records ingestion events (method, success, time, document ID)
- Tracks per-method statistics (URL-based vs file-based)
- Supports period-based queries (last_hour, last_day, last_week)

---

## Data Models & Classification Schema

### Classification Schema (`src/lib/constants.py`)

**31 Content Categories:**
Meme, Reviewers, Lyrics Pages, Festival Goers, DJs, Music Production, Music Performance, LGBTQ+, Dance, Comedic Skits, Prank Videos, Day-in-the-Life Vlogs, Fashion, Beauty, Makeup Tutorials, Family Content, Travel, Fitness, Wellness, Cooking, DIY Projects, Educational Content, Gaming, Pet & Animal Content, Home Decor, Get Ready With Me, Hauls, Social Commentary, Lip-Sync, Cinematic Storytelling, Cannot Classify

**8 Style Tags:**
Aesthetics & Vibe, Point-of-View (POV), Fast-Paced Editing, Slow Motion, Transitions, Lighting & Visual Effects (FX), Raw/Unedited, Cannot Classify

**16 Languages:**
English, Spanish, Portuguese, French, Italian, Romanian, Catalan, Arabic, Mandarin, Hindi, Russian, Japanese, German, Korean, Music Only, Unknown

**25 Music Genres (culturally organized):**
- Latin (13): Bachata, Merengue, Salsa, Cumbia, Corridos, Musica Mexicana, Reggaeton, Latin Trap, Dembow, Latin Afrobeat, Latin Electronic, Latin Pop, Hip Hop
- Non-Latin (8): Afrobeat, Reggae, Country, EDM, Hip Hop/Rap, Pop, R&B, Rock/Alternative
- Standalone (2): K-Pop, Brazilian Funk
- Special (2): No Music, Other

### Request Models

**EnhancedClassifyRequest** (v2.0, Pydantic):
```python
signedUrl: str      # Video URL (public, signed, or gs://)
mediaId: str        # Unique media identifier
soundId: str?       # Optional audio track ID
userHandle: str?    # Creator handle (auto-prefixed with @)
```

**ClassifyRequest** (v1.0, legacy):
```python
video: str          # URL or file path
video_type: str     # "url", "signed_url", or "file"
post_id: str        # TikTok post ID
handle: str         # Creator handle
timestamp: float?   # Optional seek position
```

### Response Models

**ClassificationData** (v2.0 response):
```python
category: List[str]                        # 1-3 categories
tags: List[str]                            # 0-3 style tags
video_description: str                     # 800-2500 chars
language: str                              # Detected language
common_music: List[str]                    # 0-3 music genres
unrestricted_genre: List[str]              # 1-3 creative genre tags
background_music_mode: str                 # Mood/tone descriptors
background_music_audibility: int           # 1-5 scale
music_proportion: float                    # 0.000-1.000
other_genre_description: str?              # Free text (when "Other" genre)
music_record_campaign_recommendation: str  # Yes/No/Cannot Decide
confidence_scores: Dict[str, float]        # Per-category confidence
people: str                                # Visual description
account_type: str                          # Human/Pet/Non-Human
content_purpose: str                       # Reviewer/Entertainer/Educator/Other
reviewer_specificity: str?                 # Conditional field
frame_url: str                             # GCS frame URL
vector_id: str                             # Jina document ID
processing_time_ms: float                  # Total processing time
mediaId: str                               # Echo back
userHandle: str                            # Echo back
soundId: str?                              # Echo back
```

---

## Deduplication System

The system prevents duplicate video entries using composite IDs and Jina's native upsert.

### Composite ID Format

```
pulse_video_id = "{userHandle}:{mediaId}"
Example: "@djnathii:7548978228686327096"
```

Generated by `src/lib/jina_utils.py`:
```python
generate_pulse_video_id("@djnathii", "7548978228686327096")
# → "@djnathii:7548978228686327096"
```

### Upsert Mechanism

The `id_field: "pulse_video_id"` parameter in Jina API payloads enables native upsert:
- If no document with matching `pulse_video_id` exists → **create** (insert)
- If a document with matching `pulse_video_id` exists → **update** (overwrite)

This eliminates the need for pre-query deduplication checks and handles concurrent updates with a last-write-wins strategy.

### Response Tracking

The API response includes:
- `"operation": "create"` or `"update"` — indicates whether the video was new or re-processed
- `"ingestion_count": N` — tracks how many times the video has been ingested
- `"pulse_video_id": "..."` — the composite key used

---

## Error Handling & Retry Logic

### Application-Level Error Handling

**Custom exception hierarchy** (`src/models/error_models.py`):
- `VideoClassifierException` — Base exception with error code enum
- `ValidationError` — Input validation failures (400)
- `VideoProcessingError` — Video download/extraction failures (500)

**Flask error handlers** (registered in `create_app()`):
- 400 → Bad Request
- 404 → Not Found
- 413 → Request Too Large (> 500MB)
- 500 → Internal Server Error
- `VideoClassifierException` → Maps to appropriate HTTP status

### Jina Client Retry Logic

```
Attempt 1 ──fail──▶ wait 1s ──▶ Attempt 2 ──fail──▶ wait 2s ──▶ Attempt 3 ──fail──▶ Return error
```

- **Max retries:** 3
- **Backoff:** Exponential (1s × 2^(attempt-1))
- **Retryable:** 5xx, 429, timeouts, connection errors
- **Non-retryable:** 4xx (except 429) — fail immediately

### Gemini File Cleanup Retry

After video analysis, the temporary Gemini file is deleted with retry:
- **Max retries:** 3
- **Backoff base:** 2s (exponential: 2s, 4s, 6s)

---

## Configuration & Environment

### Required Environment Variables

| Variable | Default | Purpose |
|----------|---------|---------|
| `GOOGLE_CLOUD_PROJECT` | — | GCP project ID |
| `GOOGLE_APPLICATION_CREDENTIALS` | — | Service account JSON path |
| `GCP_PROJECT_ID` | `influur` | GCP project |
| `GCP_LOCATION` | `us-central1` | Vertex AI region |
| `VERTEX_AI_MODEL` | `gemini-2.5-flash` | Gemini model name |
| `USE_VERTEX_AI` | `true` | Use Vertex AI vs direct Gemini API |
| `JINA_API_URL` | `http://34.132.139.92` | Jina server address |
| `JINA_API_PORT` | `5000` | Jina server port |
| `COLLECTION_NAME` | `pulse-videos-v2` | Jina collection name |

### Optional Environment Variables

| Variable | Default | Purpose |
|----------|---------|---------|
| `FLASK_HOST` | `0.0.0.0` | Server bind address |
| `FLASK_PORT` | `8000` | Server port |
| `FLASK_ENV` | `production` | Environment mode |
| `LOG_LEVEL` | `INFO` | Logging level |
| `CORS_ORIGINS` | `*` | Allowed CORS origins |
| `MAX_CONTENT_LENGTH` | `500MB` | Max upload size |
| `ENABLE_FRAME_UPLOAD` | `true` | Upload frames to GCS |
| `USE_SIGNED_URLS` | `true` | Generate signed URLs |
| `ENABLE_RERANKING` | `true` | Enable query reranking |
| `RERANK_TOP_K` | `20` | Reranking candidate count |
| `GCS_BUCKET_NAME` | `pulse_video_frames` | Frame storage bucket |
| `URL_EXPIRATION_SECONDS` | `3600` | Signed URL expiry |
| `LANGSMITH_API_KEY` | — | LangSmith API key |
| `LANGSMITH_PROJECT` | `pulse-rag` | LangSmith project name |
| `LANGSMITH_TRACING` | `false` | Enable LLM tracking |
| `JINA_API_KEY` | — | Optional Jina auth |
| `BATCH_BUCKET` | `pulse_batch_jobs` | Batch job GCS bucket |
