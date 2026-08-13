---
layout: post
title: Discovery Pipeline System Design
date: 2026-08-13 13:39 +0300
tags:
  - systemdesign
  - vectai
  - discovery-pipeline
---

## 1. Goal

Design the **Discovery Pipeline** - the multi-source job ingestion subsystem of the Vecta AI platform. The pipeline scrapes job postings from 8+ source types (JobSpy, Scrapling spiders, REST APIs, RSS feeds, ATS providers, manual URL import, unified composite channels, and ATS portal scans), normalizes heterogeneous scraped data into a canonical `Job` model, deduplicates, persists with trust scoring, embeds for semantic search, and matches against user profiles. It is orchestrated via Celery with real-time SSE progress streaming to the frontend.

## 2. Non-Goals (YAGNI)

- **Browser-use autofill / application submission** - handled by the separate [[Browser Engine]] subsystem.
- **LLM evaluation of job matches** - handled by the [[Evaluations]] subsystem, triggered post-match.
- **Resume parsing / optimization** - handled by the [[Resumes]] subsystem.
- **Billing / credit charging for discovery runs** - discovery is free-tier; credit accounting lives in [[AI Credits System]].
- **Workflow Engine v2** - the pipeline supports an optional `WORKFLOW_ENGINE_ENABLED` bypass, but the workflow engine itself is a separate subsystem.
- **Real-time job alerts / notifications scheduling** - the pipeline emits completion events; scheduling recurring discovery is out of scope.

## 3. Requirements

### 3.1 Functional Requirements (Top 3 Prioritized)

| Priority | ID | Requirement |
|----------|----|-------------|
| P0 | FR-01 | **Multi-source ingestion** - The system MUST ingest job postings from 8+ source types through a unified adapter interface, with each source implementing a common `fetch()` contract that returns normalized `ScrapedJob` objects. |
| P0 | FR-02 | **Canonical normalization + dedup** - Heterogeneous scraped data (HTML, JSON, RSS, DataFrames) MUST be normalized into a single canonical `Job` model and deduplicated via a `(source, source_job_id)` unique constraint before persistence. |
| P1 | FR-03 | **Real-time progress streaming** - Pipeline progress (scrape → embed → match phases) MUST be streamed to the frontend in real time via Server-Sent Events, with reconnection replay support for dropped clients. |

### 3.2 Additional Functional Requirements

| ID | Requirement |
|----|-------------|
| FR-04 | **Partial failure tolerance** - Individual source failures MUST NOT fail the entire pipeline; the run completes with `PARTIAL` status when at least one source succeeds. |
| FR-05 | **Manual single-URL import** - Users MAY import a single job posting by URL, routed through spider matching then LLM extraction fallback. |
| FR-06 | **Bulk CSV import** - Users MAY import jobs via CSV through `django-import-export` `JobResource`. |
| FR-07 | **Trust scoring** - Every ingested job MUST receive a trust score (0-100) and level (high/medium/low) based on URL validity, domain reputation, and company-domain alignment heuristics. |
| FR-08 | **Tier-based source capping** - Free-tier users are capped at 3 sources × 20 jobs/source; paid tiers are uncapped. |
| FR-09 | **Two-tier caching** - Results are cached in Redis (per-run) and the Postgres `Job` table (24h pool cache) to avoid redundant scraping. |
| FR-10 | **Stale run reaping** - Runs stuck in non-terminal states for >30 minutes MUST be automatically marked `FAILED` by a scheduled maintenance task. |
| FR-11 | **Semantic matching** - After embedding, jobs MUST be matched against the user's profile via the `SemanticMatcher` (RRF fusion of semantic + keyword scores). |
| FR-12 | **BYOK support** - API sources MUST support Bring-Your-Own-Key resolution (injected key > user BYOK key > system settings key). |

### 3.3 Non-Functional Requirements (Quantified)

| ID | Requirement | Target |
|----|-------------|--------|
| NFR-01 | **Pipeline latency (p50)** | End-to-end discovery run (scrape → match) completes in < 60 seconds for 5 sources. |
| NFR-02 | **Pipeline latency (p95)** | < 180 seconds for 10+ sources with anti-bot evasion enabled. |
| NFR-03 | **Source isolation** | A single source failure (timeout, block, crash) has zero impact on other sources' results. |
| NFR-04 | **SSE delivery latency** | Progress events reach the client within 500ms of task emission (Redis Streams + Channels). |
| NFR-05 | **Dedup accuracy** | Zero duplicate `Job` rows for identical `(source, source_job_id)` pairs across concurrent runs. |
| NFR-06 | **Availability** | Discovery queue processes tasks with 99.9% uptime (RabbitMQ consumer pool). |
| NFR-07 | **Throughput** | Support 100+ concurrent discovery runs without queue saturation (horizontal Celery worker scaling). |

### 3.4 Capacity Estimation (Just-in-Time)

| Metric | Estimate | Basis |
|--------|----------|-------|
| Jobs per run | 20–500 | `limit_per_source` (1-100) × sources (1-50) |
| Concurrent runs | 100 | One per active user session |
| Scrape task duration | 5–60s | HTTP APIs fast; Scrapling stealth spiders slow (anti-bot) |
| Embedding batch size | Configurable (`VECTOR_DB["EMBEDDING_BATCH_SIZE"]`) | Chunks of job IDs dispatched to `compute_heavy` queue |
| SSE stream TTL | 7200s (2h) | Redis stream `maxlen=200`, `ttl=7200` |
| Stale run cutoff | 30 minutes | `STALE_RUN_CUTOFF_MINUTES` |
| Daily maintenance | 1 run at 4 AM | Celery Beat schedule |

---

## 4. Core Entities

- **`DiscoveryRun`** - The central aggregate root representing one pipeline execution. Tracks immutable inputs (engine, sources, keywords, location, limits), mutable state (status, phase results), and timing. Status state machine: `PENDING → SCRAPING → EMBDENDING → MATCHING → COMPLETED | FAILED | PARTIAL`. Has M2M to `Job` via `discovered_jobs`.
- **`DiscoveryEngine`** (enum) - Acquisition channel classification: `API`, `RSS`, `PROVIDER`, `SCRAPING`, `JOBSPY`, `MANUAL`, `UNIFIED`, `PORTAL`. Drives factory dispatch.
- **`DiscoveryRunStatus`** (enum) - State machine values: `PENDING`, `SCRAPING`, `EMBEDDING`, `MATCHING`, `COMPLETED`, `FAILED`, `PARTIAL`.
- **`Job`** - The canonical persisted job posting. Unique constraint on `(source, source_job_id)`. Carries trust metadata (`trust_score`, `trust_level`, `trust_flags`), embedding status (`is_embedded`), and liveness (`liveness_status`, `last_verified`).
- **`ScrapedJob`** - Normalized in-memory representation of a scraped posting before persistence. Auto-generates `source_job_id` from URL MD5. Carries salary, job type, experience level, raw data.
- **`ElementFingerprint`** - Persists adaptive CSS selector fingerprints for Scrapling's anti-drift mechanism. Unique on `(site_key, identifier)`.
- **`RequestedJobSource`** - Tracks user-requested domains lacking a dedicated spider (for LLM fallback path). Unique on `domain`, with `request_count`.
- **`JobSource`** (ABC) - Abstract base for all source adapters. Self-registers via `__init_subclass__` into `JobSource.registry`. Defines `fetch()` contract.
- **`DiscoverySourceResult`** - Per-source aggregation of scrape results: created job IDs, cached IDs, counts, errors, timing.
- **`PipelineContext`** - State bag passed between Celery phases. Reconstructed from `DiscoveryRun` DB row at each phase boundary (the DB row is shared state, not in-memory objects).
- **`ScraplingConfig`** - Per-scrape configuration: keywords, location, proxy, headless, Cloudflare solving, stealth flags, timeouts.
- **`ChannelDef`** / **`CHANNELS`** - Curated marketplace channel definitions (remote-first, enterprise-tech, startup-vc, etc.) for unified discovery.

---

## 5. API Endpoints

### 5.1 Primary Discovery Endpoint

```
POST /api/jobs/discover/
```

**Auth:** `IsAuthenticated` (JWT cookie). Profile resolved via `ProfileMixin.get_profile()`.

**Request** (`DiscoverySerializer`):

| Field | Type | Required | Default | Notes |
|-------|------|----------|---------|-------|
| `engine` | ChoiceField | no | `scraping` | `DiscoveryEngine.choices` |
| `keywords` | CharField | **yes** | - | max 255 |
| `location` | CharField | no | `"Remote"` | |
| `limit_per_source` | IntegerField | no | 20 | 1–100 |
| `min_match_score` | IntegerField | no | 30 | 0–100 |
| `max_days_old` | IntegerField | no | null | 1–365 (Adzuna window) |
| `profile_id` | IntegerField | no | null | |
| `sources` | ListField(CharField) | no | - | min_length 1 |
| `targets` | ListField(DictField) | no | `[]` | portal/ATS targets |
| `manual_job_url` | CharField | no | null | required if engine=manual |
| `proxy` | CharField | no | null | Scrapling-only |
| `headless` | BooleanField | no | true | Scrapling-only |

**Validation rules:**
- `engine == "manual"` → requires `manual_job_url`
- `engine == "portal"` → requires non-empty `targets`
- All other engines → require non-empty `sources`

**Response:** `HTTP 202 Accepted`
```json
{"success": true, "engine": "scraping", "message": "...", "run_id": "uuid"}
```

The `run_id` is the stable SSE stream key for progress subscription.

### 5.2 SSE Progress Stream

```
GET /api/jobs/discover/stream/<run_id>/
```

**Auth:** `IsAuthenticated`. Ownership check: run must belong to the caller's active profile.

**Behavior:**
- Fast-path terminal events: if run is already `COMPLETED`/`PARTIAL`/`FAILED`, emits synthetic terminal event immediately.
- Supports `Last-Event-ID` header / `last_id` query param for reconnection replay.
- Heartbeat interval: 20s. Timeout: 600s (`DISCOVERY_SSE_TIMEOUT_SECONDS`).

**Event types:** `pipeline_started`, `source_started`, `source_completed`, `source_failed`, `phase_started`, `phase_completed`, `completed`, `failed`, `progress`.

### 5.3 CSV Import / Export

```
POST /api/jobs/import/     (via JobResource)
GET  /api/jobs/export/     (via JobResource)
```

`django-import-export` `ModelResource` for `Job`. Import deduplicates on `url`. After save, adds job to `discovery_run.discovered_jobs` and upserts a `JobMatch` (score 100, status `queued`, origin `csv_import`).

### 5.4 ATS Scan Endpoints

```
POST /api/jobs/discover/ats-scan/          (single company URL)
POST /api/jobs/discover/ats-aggregator/    (all registered aggregators)
```

Standalone tasks that create a lightweight `DiscoveryRun`, scan via `ATSScanService`, and persist jobs.

---

## 6. Data Flow

The pipeline executes five sequential phases, with phases 1 (scrape) and 3 (embed) internally parallelized via Celery chords.

```
Phase 1: SCRAPE (parallel across sources)
─────────────────────────────────────────
API request → DiscoveryRun (PENDING)
          → orchestrate_discovery (dispatcher)
              → prepare_and_cap_sources()  [resolve channels, apply tier caps]
              → for each source: discover_source_task.s(run_id, source_key)
                  → check Redis result cache
                  → cache miss → get_engine_class() → JobSourceEngine.run(source)
                      → SourceRegistry.get(source_key) → JobSource subclass
                      → source.from_run(run)  [resolve BYOK, targets, config]
                      → source.fetch(keywords, location, limit) → list[ScrapedJob]
                      → check 24h pool cache (_fetch_from_pool)
                      → [asdict(j) for j in jobs]
                  → return DiscoverySourceResult (never raises)
              → [chord: parallel scrape tasks]

Phase 2: AGGREGATE
──────────────────
aggregate_scrape_results (chord callback)
  → iterate per-source results
  → tally stats, collect all_job_ids
  → if total_saved == 0 → FAILED
  → else → status EMBEDDING, return list(job_ids)

Phase 3: EMBED (parallel across batches)
────────────────────────────────────────
fan_out_embed
  → chunk job_ids by EMBEDDING_BATCH_SIZE
  → for each chunk: embed_jobs_batch_task.s(chunk)  [compute_heavy queue]
  → [chord: parallel embed batches]
  → aggregate_embed_results
      → sum {total, embedded, errors}
      → status MATCHING

Phase 4: MATCH
──────────────
match_task
  → SemanticMatcher(profile, discovery_run).match_and_save(min_score, topk)
  → determine terminal status: COMPLETED or PARTIAL
  → persist match_stats, total_matches, completed_at

Phase 5: FINALIZE
─────────────────
DiscoveryNotifier.on_complete(ctx)
  → actstream action + push notification
  → SSE report_complete (progress=100)
```

**Manual URL path** (bypasses Phase 1 parallel scrape):
```
POST /api/jobs/discover/ (engine=manual)
  → process_manual_job_link
      → spider routing by domain (JobSource.registry)
      → spider match → process_with_spider (Scrapling)
      → no spider → LLM fallback (fetch_html → Gemini JSON extraction)
      → ResultNormalizer.element_to_job → Job.objects.update_or_create(url=...)
      → chain: fan_out_embed → match_task → finalize_manual_pipeline
```

---

## 7. High-Level Design

### 7.1 Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              FRONTEND (React)                               │
│                                                                             │
│   POST /api/jobs/discover/          GET /api/jobs/discover/stream/<run_id>/ │
│        │                                        ▲                          │
│        │ 202 Accepted                           │ SSE Events                │
│        │ (run_id)                               │ (progress, phases)        │
└────────┼────────────────────────────────────────┼───────────────────────────┘
         │                                        │
┌────────▼────────────────────────────────────────┼───────────────────────────┐
│                     DJANGO REST FRAMEWORK       │                           │
│                                                  │                           │
│  ┌─────────────────────┐    ┌──────────────────┐ │  ┌─────────────────────┐ │
│  │   JobViewSet        │    │  DiscoverySerial │ │  │  discovery_stream   │ │
│  │   .discover()       │───▶│  izer (validate) │ │  │  _view (SSE)        │ │
│  │   jobs.py:184       │    │  serializers.py  │ │  │  sse.py:35          │ │
│  └─────────┬───────────┘    └──────────────────┘ │  └─────────┬───────────┘ │
│            │                                      │            │             │
│            ▼                                      │            ▼             │
│  ┌───────────────────────────────────┐            │  ┌─────────────────────┐│
│  │     Celery Task Dispatcher        │            │  │  generic_sse_gen    ││
│  │                                   │            │  │  core/sse.py:59     ││
│  │  orchestrate_discovery            │            │  │  (replay + live)    ││
│  │  process_manual_job_link          │            │  └─────────┬───────────┘│
│  │  run_ats_url_scan                 │            │            │             │
│  └───────────┬───────────────────────┘            │            │             │
└──────────────┼────────────────────────────────────┼────────────┼─────────────┘
               │                                    │            │
┌──────────────▼────────────────────────────────────┼────────────┼─────────────┐
│                     CELERY WORKERS                │            │             │
│                   (job_discovery queue)           │            │             │
│                                                   │            │             │
│  ┌─────────────────────────────────────────────┐  │            │             │
│  │         Canvas Pipeline (chain + chord)      │  │            │             │
│  │                                             │  │            │             │
│  │  group ──┬── discover_source_task(src₁)     │  │            │             │
│  │          ├── discover_source_task(src₂)     │  │            │             │
│  │          └── discover_source_task(srcₙ)     │  │            │             │
│  │               │                             │  │            │             │
│  │               ▼  [chord callback]           │  │            │             │
│  │          aggregate_scrape_results           │  │            │             │
│  │               │                             │  │            │             │
│  │               ▼                             │  │            │             │
│  │          fan_out_embed ──► chord ──┬── embed │  │            │             │
│  │               │                    │  batch₁ │  │            │             │
│  │               │                    ├── embed  │  │            │             │
│  │               │                    │  batch₂ │  │            │             │
│  │               │                    └── ...    │  │            │             │
│  │               │                       │      │  │            │             │
│  │               │                       ▼      │  │            │             │
│  │               │              aggregate_embed │  │            │             │
│  │               │                             │  │            │             │
│  │               ▼                             │  │            │             │
│  │          match_task (terminal)              │  │            │             │
│  └─────────────────────────────────────────────┘  │            │             │
│                                                   │            │             │
└───────────────────────────────────────────────────┼────────────┼─────────────┘
                                                    │            │
┌───────────────────────────────────────────────────┼────────────┼─────────────┐
│                   SERVICE LAYER                   │            │             │
│                                                   │            │             │
│  ┌──────────────────┐  ┌───────────────────────┐  │            │             │
│  │  SourceRegistry  │  │   JobSourceEngine     │  │            │             │
│  │  (runtime lookup)│  │   (universal engine)  │  │            │             │
│  │  registry.py     │  │   job_source_engine   │  │            │             │
│  └────────┬─────────┘  └───────────┬───────────┘  │            │             │
│           │                        │              │            │             │
│           ▼                        ▼              │            │             │
│  ┌────────────────────────────────────────────┐   │            │             │
│  │         JobSource Adapter Hierarchy         │   │            │             │
│  │                                            │   │            │             │
│  │  APISource    ScraplingJobSource           │   │            │             │
│  │  ATSJobSource JobSpyJobSource              │   │            │             │
│  │  PortalJobSource RSSSource                 │   │            │             │
│  │       │             │                      │   │            │             │
│  │       ▼             ▼                      │   │            │             │
│  │  27 APIs     18 Scrapling spiders          │   │            │             │
│  │  7 providers 27 portal handlers            │   │            │             │
│  │  6 RSS feeds 2 JobSpy strategies           │   │            │             │
│  └────────────────────────────────────────────┘   │            │             │
│                                                   │            │             │
│  ┌──────────────────┐  ┌───────────────────────┐  │            │             │
│  │ ResultNormalizer │  │   JobPersister        │  │            │             │
│  │ (raw→ScrapedJob) │  │   (bulk upsert)       │  │            │             │
│  └──────────────────┘  └───────────────────────┘  │            │             │
│                                                   │            │             │
│  ┌──────────────────┐  ┌───────────────────────┐  │            │             │
│  │ TrustValidator   │  │  AdaptiveStorage      │  │            │             │
│  │ (score/flags)    │  │  (selector fingerprints│  │            │             │
│  └──────────────────┘  └───────────────────────┘  │            │             │
│                                                   │            │             │
└───────────────────────────────────────────────────┼────────────┼─────────────┘
                                                    │            │
┌───────────────────────────────────────────────────┼────────────┼─────────────┐
│                     DATA LAYER                    │            │             │
│                                                   │            │             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────┐ │  ┌────────▼──────────┐ │
│  │  PostgreSQL  │  │    Redis     │  │  Qdrant  │ │  │  Redis Streams    │ │
│  │              │  │              │  │          │  │  │  + Channels       │ │
│  │  DiscoveryRun│  │  Result cache│  │  Job     │ │  │                   │ │
│  │  Job         │  │  24h pool    │  │  vectors │ │  │  jc:events:       │ │
│  │  JobMatch    │  │  CB state    │  │          │ │  │  discovery:{run}  │ │
│  │  ElementFp   │  │              │  │          │ │  │                   │ │
│  └──────────────┘  └──────────────┘  └──────────┘ │  └───────────────────┘ │
│                                                   │                        │
└───────────────────────────────────────────────────┴────────────────────────┘
```

### 7.2 Component Responsibilities

| Component | Responsibility |
|-----------|----------------|
| `JobViewSet.discover()` | API entry point. Validates request, creates `DiscoveryRun`, dispatches Celery task, returns 202. |
| `orchestrate_discovery` | Lightweight dispatcher. Resolves sources, applies tier caps, builds Canvas pipeline, fires it. |
| `discover_source_task` | Per-source scraper. Never raises - returns error dict so chord always fires. |
| `aggregate_scrape_results` | Chord callback. Tallies per-source stats, determines terminal vs. continue. |
| `fan_out_embed` | Chunks job IDs, builds embed chord, substitutes itself via `self.replace()`. |
| `match_task` | Terminal phase. Runs semantic matching, determines `COMPLETED` vs `PARTIAL`. |
| `SourceRegistry` | Runtime source lookup. Derives from `JobSource.registry` (populated by side-effect import). |
| `JobSourceEngine` | Universal engine. Template method: acquire → persist → post-persist. |
| `JobSource` (ABC) | Adapter contract. `fetch()` → `list[ScrapedJob]`. Self-registers via `__init_subclass__`. |
| `ResultNormalizer` | Raw HTML/text → `ScrapedJob`. Salary parsing, date parsing, URL normalization. |
| `JobPersister` | Bulk upsert to `Job`. Dedup by `(source, source_job_id)`. `bulk_create(update_conflicts=True)`. |
| `TrustValidator` | Heuristic scoring. URL validity, domain reputation, company-domain alignment. Never drops jobs. |
| `AdaptiveStorage` | Selector fingerprint persistence (Django ORM or Redis) for Scrapling anti-drift. |
| `ProgressReporter` | Unified progress emitter. Writes Celery task state + Redis SSE + profile pipeline stream. |
| `EventStreamService` | Redis Streams + Channels transport. `XADD` + `group_send`. Replay support. |
| `discovery_stream_view` | SSE endpoint. Ownership check, terminal fast-path, replay + live loop. |

---

## 8. Deep Dives

### 8.1 Deep Dive: Multi-Source Ingestion Gateway

**Problem:** The platform must ingest job postings from 8+ fundamentally different source types - REST APIs (Adzuna, JSearch), RSS feeds (Remotive, HN), HTML spiders (Scrapling: Indeed, LinkedIn), DataFrame strategies (JobSpy), ATS providers (Greenhouse, Ashby), and portal scanners (Workday, BambooHR). Each has different auth, pagination, rate limits, response formats, and anti-bot measures.

**Solution: Polymorphic adapter hierarchy with auto-registration.**

```
JobSource (ABC)
├── APISource           - REST/GraphQL with BYOK resolution, header injection
├── ATSJobSource        - ATS providers with dual identity (registry + URL detector)
├── ScraplingJobSource  - Scrapling Spider + JobSource multiple inheritance
├── JobSpyJobSource     - DataFrame-based strategies with clean_dataframe()
├── PortalJobSource     - ATS portal handlers with registry key prefixing
└── RSSSource           - feedparser-based with entry parsing
```

**Auto-registration via `__init_subclass__`:** Defining a subclass with a `key` attribute automatically adds it to `JobSource.registry[key]`. No explicit registration call, no DB migration, no config change. Adding a new source = creating one file.

**Key design decisions:**
1. **Single universal engine** - All engine categories (`scrapling`, `jobspy`, `portal`, `api`, `rss`, `providers`, `unified`) resolve to `JobSourceEngine`. The `ENGINE_REGISTRY` is a flat map, not a class hierarchy.
2. **`fetch()` contract** - Every source implements `fetch(*, keywords, location, limit) → list[ScrapedJob]`. The engine never inspects source internals.
3. **Polymorphic factory** - `source.from_run(run)` lets sources reconstruct state (BYOK keys, targets, config) from the `DiscoveryRun` row.
4. **Post-persist hook** - `source.on_post_persist(run, result)` lets sources emit notifications or update metrics after their jobs are saved.

**BYOK resolution precedence** (in `APISource`): injected `api_key` > user's `UserApiKey` (matched by `byok_provider`) > system settings key. This enables per-user API key overrides without code changes.

**Trade-offs:**
- **Pro:** Adding a source requires zero changes to existing code (open/closed principle).
- **Con:** The registry is a global mutable class variable - import order matters, and testing requires care to avoid cross-test contamination.
- **Mitigation:** `SourceRegistry._registry()` forces a fresh import on first access and catches `ImportError` gracefully (missing optional deps like `scrapling`/`jobspy` don't crash the whole registry).

### 8.2 Deep Dive: Deduplication Strategy

**Problem:** The same job posting may appear across multiple sources (e.g., a role posted on both Indeed and LinkedIn), be re-scraped across runs, or be imported via both CSV and spider. We must avoid duplicate `Job` rows while allowing the same role from different sources to coexist.

**Solution: Composite unique key `(source, source_job_id)` with multi-layer dedup.**

**Layer 1 - In-batch dedup (within a single `discover_source_task`):**
```python
# JobPersister.persist()
seen: set[tuple[str, str]] = set()
for job in jobs_data:
    source = SourceRegistry.resolve_source_value(source_key)
    source_job_id = job.get("source_job_id") or md5(job["url"])[:255]
    key = (source, source_job_id)
    if key in seen:
        continue
    seen.add(key)
```

**Layer 2 - Database unique constraint:**
```python
# Job model Meta
UniqueConstraint(fields=["source", "source_job_id"], name="unique_job_per_source")
```

**Layer 3 - Upsert on conflict:**
```python
Job.objects.bulk_create(
    job_instances,
    batch_size=500,
    update_conflicts=True,
    unique_fields=["source", "source_job_id"],
    update_fields=["title", "company", "description", "salary_min", ...],
)
```
When a duplicate is detected, the existing row is **updated** (not skipped), ensuring fresh data (e.g., updated description, re-verified liveness).

**Layer 4 - URL-based dedup for manual import:**
```python
# process_manual_job_link
Job.objects.update_or_create(url=job_url, defaults={...})
```

**Layer 5 - ScrapedJob-level URL dedup:**
```python
# ResultNormalizer.deduplicate_jobs()
# URL-based dedup preserving order (for LLM-extracted results)
```

**`source_job_id` generation:**
- APIs: use the provider's native ID (e.g., Adzuna `job["id"]`).
- Scraplers: use the job's canonical URL or a provider-specific ID field.
- Fallback: MD5 of the job URL, truncated to 255 chars.

**Trade-offs:**
- **Pro:** The composite key allows the same role from different sources to coexist (different `source` values).
- **Pro:** Upsert-on-conflict means re-scraping refreshes data rather than silently skipping.
- **Con:** `source_job_id` collisions can occur if a provider reuses IDs across different job boards - mitigated by the `source` prefix.
- **Con:** MD5-based fallback IDs are stable but opaque; debugging requires URL inspection.

### 8.3 Deep Dive: Pipeline Orchestration with Partial Failure

**Problem:** A discovery run may scrape 10+ sources. Some will fail (timeouts, anti-bot blocks, API rate limits, site redesigns). The pipeline must not fail entirely because of one bad source - users expect to see whatever jobs were successfully scraped.

**Solution: Chord-level failure isolation via error-dict contract.**

The key design decision is that **`discover_source_task` never raises at the Celery level**. It catches all exceptions and returns a serialized error dict:

```python
# discover_source.py
try:
    jobs = engine.run(source=source_key)
except SoftTimeLimitExceeded:
    result.error = "SoftTimeLimitExceeded: timed out"
except Exception as exc:
    result.error = f"{type(exc).__name__}: Spider failed for {source_key} -> {exc}"
finally:
    return asdict(result)  # Always returns, never raises
```

This ensures the **chord callback always fires** - Celery chords require all header tasks to complete (success or failure). By returning an error dict instead of raising, we guarantee the chord progresses.

**Aggregation logic in `aggregate_scrape_results`:**
```python
for result in scrape_results:
    if result.get("error"):
        # Record error stats, do not contribute job IDs
        source_stats[source] = {"error": result["error"], "saved": 0}
    elif result.get("from_cache"):
        # Forward cached job IDs
        all_job_ids.update(result["cached_job_ids"])
    else:
        # Fresh scrape
        all_job_ids.update(result["created_job_ids"])

if total_saved == 0:
    run.status = FAILED  # All sources failed
else:
    run.status = EMBEDDING  # At least one source succeeded
```

**Terminal status determination in `match_task`:**
```python
if any(source_stats[s].get("error") for s in source_stats):
    run.status = PARTIAL  # Some sources failed, but we have jobs
else:
    run.status = COMPLETED  # All sources succeeded
```

**Chain break on empty jobs:**
```python
# fan_out_embed
if not job_ids:
    run.status = FAILED
    self.request.chain = None  # Breaks the chain - match_task is ABORTED
    return zeroed_stats
```

**Stale run safety net:**
```python
# cleanup_stale_discovery_runs (daily 4 AM)
DiscoveryRun.objects.filter(
    status__in=[PENDING, SCRAPING, EMBDENDING, MATCHING],
    started_at__lt=now() - timedelta(minutes=30),
).update(status=FAILED, completed_at=now())
```

**Trade-offs:**
- **Pro:** Partial failure is the default - users always see what succeeded.
- **Pro:** The `PARTIAL` status is surfaced to the frontend via SSE (`partial: true` flag), so the UI can show a warning.
- **Con:** Error dicts are untyped - the aggregation logic relies on string parsing of `result["error"]`.
- **Mitigation:** `DiscoverySourceResult` dataclass provides structure; errors are logged with full context for debugging.

### 8.4 Deep Dive: Circuit Breaker Pattern

**Problem:** Scrapling spiders and ATS providers can enter failure cascades - a site redesign breaks a spider, or an API starts rate-limiting aggressively. Retrying immediately wastes resources and can get IPs banned.

**Solution: Per-source circuit breaker with Redis-backed state.**

**Note:** The circuit breaker in this codebase (`vectai/browser_manager/circuit_breaker.py`) operates at the **browser autofill layer** (Playwright strategies), not the scraping layer. The discovery scraping layer handles failures via different mechanisms:

**Scrapling anti-bot resilience (`ScraplingJobSource`):**
```python
BLOCKED_STATUS_CODES = {401, 403, 407, 429, 444, 500, 502, 503, 504}

async def is_blocked(self, response) -> bool:
    if response.status in self.BLOCKED_STATUS_CODES:
        return True
    text = (await response.text()).lower()
    if "access denied" in text or "rate limit" in text:
        return True
    return False

async def retry_blocked_request(self, request, response):
    request.sid = "stealthy"  # Switch to stealthy session
    return request
```

**Provider-level retry with backoff (`AshbyProvider`, `PortalJobSource`):**
```python
ASHBY_RETRIES = 2
for attempt in range(ASHBY_RETRIES + 1):
    if attempt > 0:
        backoff = 1_000 * 2 ** (attempt - 1) + secrets.randbelow(500)
        time.sleep(backoff / 1000.0)
    # ... httpx GET
```

**Task-level retry policy:**
- `orchestrate_discovery`: `max_retries=2`, `retry_backoff=True` (safe - only dispatches).
- `discover_source_task`: `max_retries=1`, `default_retry_delay=15`.
- Chord callbacks (`aggregate_scrape_results`, `fan_out_embed`, `match_task`): **no retries** - terminal/aggregator phases.

**Adaptive selector persistence (`AdaptiveStorage`):**
```python
# DjangoAdaptiveStorage - backed by ElementFingerprint rows
def save(self, element, identifier):
    ElementFingerprint.objects.update_or_create(
        site_key=self.site_key,
        identifier=identifier,
        defaults={"fingerprint": _StorageTools.element_to_dict(element)},
    )

def retrieve(self, identifier) -> dict | None:
    row = ElementFingerprint.objects.only("fingerprint").filter(
        site_key=self.site_key, identifier=identifier
    ).first()
    return row.fingerprint if row else None
```
When a site redesign breaks selectors, the adaptive system falls back to stored fingerprints from previous successful crawls, reducing the blast radius of DOM changes.

**Trade-offs:**
- **Pro:** Per-source failure isolation prevents cascade failures.
- **Pro:** Adaptive selectors provide resilience against minor DOM changes without code deploys.
- **Con:** The circuit breaker is not yet integrated into the scraping layer (only browser autofill). Scraping failures rely on per-task retries and error-dict contracts.
- **Future consideration:** A discovery-layer circuit breaker (per source key, Redis-backed) could skip known-failing sources for a cooldown period, reducing wasted scrape time.

### 8.5 Deep Dive: SSE Progress Streaming

**Problem:** Discovery runs take 30-180 seconds. Users need real-time feedback on which phase is running, which sources succeeded/failed, and when results are ready - without polling.

**Solution: Redis Streams + Django Channels → SSE with reconnection replay.**

**Architecture:**
```
Celery Task ──► ProgressReporter ──► publish_progress(run_id, payload)
                                        │
                                        ▼
                              EventStreamService.publish()
                                        │
                                        ├── XADD to Redis stream
                                        │   (jc:events:discovery:{run_id})
                                        │   maxlen=200, ttl=7200s
                                        │
                                        └── group_send to Channels layer
                                            (discovery_{run_id} group)
                                                  │
                                                  ▼
                              ┌─────────────────────────────────┐
                              │  discovery_stream_view (SSE)     │
                              │  sse.py:35                       │
                              │                                  │
                              │  1. resolve_active_profile()     │
                              │  2. ownership check (404 if not) │
                              │  3. terminal fast-path           │
                              │  4. generic_sse_generator()      │
                              │     ├── replay missed events     │
                              │     └── live receive loop        │
                              └─────────────────────────────────┘
```

**Event types emitted by `ProgressReporter`:**

| Event | When | Payload |
|-------|------|---------|
| `pipeline_started` | Orchestrator dispatches | `{sources, progress: 5}` |
| `source_started` | Per-source scrape begins | `{source, progress: 10-30}` |
| `source_completed` | Per-source scrape succeeds | `{source, saved, progress}` |
| `source_failed` | Per-source scrape errors | `{source, error, progress}` |
| `phase_started` | Aggregation/embed/match begins | `{phase, progress}` |
| `phase_completed` | Phase finishes | `{phase, stats, progress}` |
| `completed` | Run finishes (all sources OK) | `{total_saved, total_matches, progress: 100}` |
| `failed` | Run finishes (all sources failed) | `{error, progress}` |

**Progress percentage mapping:**
- Pipeline start: 5%
- Scraping: 10-35% (per-source increments)
- Aggregation: 35-45%
- Embedding: 50-80%
- Matching: 85-100%

**Reconnection replay:**
```python
# generic_sse_generator (core/sse.py:59)
if last_event_id:
    # Phase 1: Replay stored events the client missed
    async for event in stream.areplay_after(last_event_id):
        yield event

# Phase 2: Live receive loop
while True:
    message = await channel_layer.receive(channel, timeout=heartbeat)
    if is_terminal(message):
        return
    yield message
```

The client sends `Last-Event-ID` header on reconnect. The server replays all events after that ID from the Redis stream, then transitions to live mode. This handles network drops, tab refreshes, and Celery worker restarts transparently.

**Terminal fast-path:**
```python
# If run is already COMPLETED/PARTIAL/FAILED when client connects,
# emit a synthetic terminal event immediately so the client doesn't wait.
if run.status in (COMPLETED, PARTIAL, FAILED):
    yield make_terminal_event(run)
    return
```

**Dual-write to Celery task state:**
```python
# ProgressReporter also writes to Celery task state
self.task.update_state(state="PROGRESS", meta=payload)
```
This provides a non-SSE fallback - the frontend can poll task state if SSE is unavailable (e.g., corporate proxies blocking SSE).

**Trade-offs:**
- **Pro:** Sub-500ms event delivery (Redis Streams + Channels is in-process for the Django server).
- **Pro:** Reconnection replay handles unreliable networks transparently.
- **Pro:** Dual-write (SSE + Celery state) provides a polling fallback.
- **Con:** Redis Streams retention (`maxlen=200`) means very long runs may lose early events - acceptable since the client connects at run start.
- **Con:** The heartbeat (20s) adds slight overhead per connected client - mitigated by Channels' efficient group management.

---

## 9. Resolved Tricky Points

1. **Chord callback starvation:** If a `discover_source_task` raised an exception, the chord callback would never fire and the run would hang until stale reaping. **Resolved:** Error-dict contract - tasks never raise.

2. **Shared state across Celery workers:** In-memory objects can't cross process boundaries. **Resolved:** The `DiscoveryRun` DB row is the single source of truth. Each task reconstructs `PipelineContext.from_run()`.

3. **`source_job_id` collisions across providers:** Different providers might reuse IDs. **Resolved:** Composite key `(source, source_job_id)` namespaces by source.

4. **Free-tier abuse:** Users could request 100 sources × 100 jobs. **Resolved:** `prepare_and_cap_sources()` enforces caps before dispatch (free: 3 sources × 20 jobs).

5. **SSE client reconnection after network drop:** Client misses events during disconnect. **Resolved:** Redis stream replay via `Last-Event-ID`.

6. **Scrapling import-time side effects:** Importing `vectai.discovery.sources` triggers heavy imports (camoufox, jobspy). **Resolved:** Lazy imports in `FetcherFactory.get_fetcher_class()` and `SourceRegistry._registry()`.

7. **Threshold override for "show all" users:** A user setting `min_match_score=0` (show all) was being overridden by the default. **Resolved:** Explicit `is not None` check in `get_run_match_threshold()`.

---

## 11. Cross-References

- [[Application Lifecycle & Browser Engine]] - downstream consumer of discovered jobs
- [[Evaluations]] - triggered post-match for top job matches
- [[AI Credits System]] - credit accounting (discovery is free-tier)
