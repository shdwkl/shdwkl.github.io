---
layout: post
title: Browser Infrastructure - System Design
date: 2026-08-30 15:28 +0300
scope: 
- Chrome-as-a-Service grid
- local pool fallback
- CDP plumbing
- discovery scraping
- anti-bot hardening  
---

## Phase 1 — Problem & Context

### The problem
Two consumers need real Chrome:

1. **Autofill (`HybridAutoFillEngine`)** — Playwright POM scripts (13 ATS) + LLM `BrowserUseAgent` must drive a real DOM (file inputs, shadow DOM, multi-step flows). Needs authenticated `storage_state` (cookies/localStorage) per `(profile, platform)`.
2. **Discovery (`ScraplingJobSource`)** — Job boards (ZipRecruiter, Indeed, Greenhouse…) are increasingly Cloudflare Turnstile-protected. Plain HTTP fetchers get `403 / challenge-platform` and `fingerprint/observe` blocks.

Running Chrome inside every celery worker (`--headless=new` per slot) is memory-heavy (300–600 MB per instance), slow to cold-start (1–3 s), and shares the worker's fate on OOM. Anti-bot pages also hang for 150–290 s (prod: `e406…` 289 s, `c146…` 160 s) holding a slot and then getting killed by `SIGTERM` on deploy.

### The solution
A **standalone Chrome grid** (`ghcr.io/browserless/chromium:v2.52.2`) accessed over CDP/WebSocket, with a **concurrency-bounded local pool as fallback**. Consumers never spawn Chrome directly in prod — they `connect_over_cdp(ws://browserless:3001?token=…)` through a single code path (`DualBrowserPoolManager.acquire` → `BrowserLauncher._run_session`). Scrapling discovery reuses the same grid via `AsyncStealthySession(solve_cloudflare=True)`.

---

## Phase 2 — Requirements

### Functional

| ID | Requirement |
|----|-------------|
| **FR-1** | Autofill and discovery MUST share one CDP endpoint type (`ws://` / `wss://` with token) without code forks. |
| **FR-2** | Local Chrome pool MUST be bypassed entirely when `BROWSER_CDP_URL` is healthy; fallback only when unreachable and `BROWSER_CDP_FALLBACK_TO_LOCAL=True` (`config/settings/base.py:1015`). |
| **FR-3** | Per-user BYO Browser (`BrowserProvider`: Browserless / Browserbase / Steel / Custom CDP) MUST route that user's runs to their own endpoint (`vectai/browser_manager/services/cdp_validator.py:109`). |
| **FR-4** | Custom CDP URLs MUST be validated for SSRF (`validate_cdp_url`) — reject `169.254.169.254`, `*.internal`, `localhost`, private/loopback IPs. |
| **FR-5** | Discovery scraping MUST recover from Cloudflare Turnstile without holding a grid slot indefinitely. |

### Non-functional (quantified)

| ID | Target | Source |
|----|--------|--------|
| **NFR-1** | Grid `/pressure` availability ≥ 99.5 %; health check every 30 s | `docker-compose.production.yml:346` |
| **NFR-2** | Autofill slot hold ≤ 120 s p95 (was 160–289 s before fix) | `.envs/.production/.browserless:28` `TIMEOUT=120000`, `celery` `time_limit=600` |
| **NFR-3** | Single slot memory 300–600 MB; grid `limits.memory 4Gi` ⇒ `CONCURRENT=5` safe, `10` → OOM | `compose/production` + `k8s/base/deployment-browserless.yaml:28` |
| **NFR-4** | Discovery per-source soft timeout 120 s / hard 180 s, `rate_limit 20/m` (1 per 3 s) | `vectai/discovery/tasks/discover_source.py:18` |
| **NFR-5** | Scrapling blocked retries ≤ 2 (stealthy once, then fail fast) | `config/settings/base.py:1276` `SCRAPLING_MAX_BLOCKED_RETRIES=2` |
| **NFR-6** | Logs clean ANSI-free, scrapable by OTel → Loki | `.envs/.production/.browserless:69` `DEBUG=browserless*` `DEBUG_COLORS=false` |

---

## Phase 3 — Capacity & Estimation

| Metric | Value | Derivation |
|--------|-------|------------|
| Grid capacity | `5` concurrent chromes | `.envs/.production/.browserless:19` / k8s `limits.memory 4Gi` |
| Slot memory | 300–600 MB | Chromium headless measured |
| Total grid RAM | ~1.5–3 GiB active + 1 GiB reservation | 5 × 450 MB avg |
| CPU | `2.0` cores, `shm_size 2gb` for `/dev/shm` | compose + k8s |
| Discovery burst | `len(sources)` parallel `discover_source_task` group → `aggregate` chord | single `DiscoveryRun` fans out to 5–10 sources |
| Grid slots vs burst | 5 slots / 10 sources → 5 run immediately, 5 queue (`QUEUED=10`), rest `429` after `QUEUE_TIMEOUT 30s` | `CONCURRENT/QUEUED` |
| Throughput (reuse) | 4 celery workers × `rate 20/m` ≈ 80 source scrapes / min, each 5–30 s actual | `discover_source_task` `rate_limit` |
| Cold start saving | `POOL_MIN=2` pre-boots 2 chromes (avoids 0.3–1 s `/_wait_for_cdp`) | `.envs/.production/.browserless:41` |

Bottleneck is **grid slots**, not Celery. Sustained `queued>3` or `recentlyRejected>0` in `GET /pressure` ⇒ raise `CONCURRENT` (and memory) or add `dns` sharding / `--scale browserless=N` (maps to `Deployment.replicas: N` in K8s).

---

## Phase 4 — Core Entities & Config

### Entities

| Entity | Location | Purpose |
|--------|----------|---------|
| `browserless` container | `docker-compose.production.yml:325` / `k8s/base/deployment-browserless.yaml` | Standalone `ghcr.io/browserless/chromium:v2.52.2` CDP grid. `shm_size 2gb`, `PORT 3001`, token auth. |
| `celeryworker_browser` | `docker-compose.production.yml:207` `CELERY_WORKER_TYPE=browser 6Gi` | Queue `browser_engine` (`exchange vectai topic browser.#` + DLQ) — runs `browser.*` tasks via `BrowserLauncher`. |
| `celeryworker_discovery` | `docker-compose.production.yml:247` `3Gi` | Queue `job_discovery` (`discovery.#`) — runs `discovery.*` via Scrapling. |
| `DualBrowserPoolManager` | `vectai/browser_manager/services/dualbrowserpool.py:113` `worker_browsers` singleton | Manages fixed `ChromeSlot` pool + optional Lightpanda; exports `acquire(browser_type) -> str` CDP URL. |
| `BrowserLauncher` | `vectai/browser_manager/services/browser_launcher.py:78` | `connect_over_cdp` + `playwright-stealth` + `storage_state` injection + `DOMHelper` callback. |
| `BrowserSession` | `browser_manager/models.py` | Encrypted `storage_state` per `(profile, platform)` — hydrated into Playwright contexts and `browser-use`'s temp file for CDP watch. |
| `ScraplingJobSource` | `vectai/core/discovery/sources/scrapling_source.py:78` | `Spider` base for scrapers; `is_blocked`/`retry_blocked_request` + `AsyncStealthySession(cdp_url=BROWSER_CDP_URL, solve_cloudflare=True)`. |
| `validate_cdp_url` / `resolve_user_browser_cdp_url` | `vectai/browser_manager/services/cdp_validator.py:62/109` | SSRF guard for BYO custom CDP + provider routing (`browserless/browserbase/steel`). |

### Env & Settings (prod)

```
# .envs/.production/.browserless
PORT=3001  TOKEN=<hex32>  CONCURRENT=5 QUEUED=10
TIMEOUT=120000  QUEUE_TIMEOUT=30000
PRE_BOOT_CHROME=true POOL_MIN=2
HEALTH=true EXIT_ON_HEALTH_FAILURE=true MAX_CPU_PERCENT=85 MAX_MEMORY_PERCENT=85
DEBUG=browserless*  DEBUG_COLORS=false  LOG_FORMAT=json (ignored — see §6.9)
TZ=Africa/Cairo
```

```
# .envs/.production/.django (scraping section)
BROWSER_CDP_URL=ws://browserless:3001?token=<TOKEN>  BROWSER_CDP_FALLBACK_TO_LOCAL=True
SCRAPLING_HEADLESS=true MAX_PAGES=10 SCRAPLING_MAX_BLOCKED_RETRIES=2
SCRAPLING_ENABLE_ANTI_BOT=true SCRAPLING_STEALTH_*=true
```

`config/settings/base.py:1014` `BROWSER_CDP_URL` empty = local pool only. `AUTOFILL_ENGINE` mirrors pool + breaker + auto-patch tunables (`BROWSER_POOL_SIZE=4`, `BROWSER_ENABLE_LIGHTPANDA=False`, `CIRCUIT_BREAKER_FAIL_MAX=7`).

### Celery topology

```
Exchange vectai (topic, durable)
  browser_engine  ← keys browser.#     → celeryworker_browser  (acks_late, DLQ 7d)
  job_discovery   ← keys discovery.#   → celeryworker_discovery
  compute_heavy   ← keys compute.#     → celeryworker_compute
  notifications   ← keys notifications.# 
  vectordb_writes ← vectordb.write.#   → + DLQ
```

`config/settings/base.py:438` `CELERY_TASK_QUEUES`, `CELERY_TASK_ROUTES` maps `browser.*`/`discovery.*`.

---

## Phase 5 — High-Level Design

{% include drawings/browser-infrastructure-light.svg %}

**One code path:** `worker_browsers.acquire(BROWSER_TYPE) -> str` yields `settings.BROWSER_CDP_URL` when `await _is_browserless_healthy()` succeeds; otherwise (and only if `fallback=True`) lazily `initialize()` the local Chrome pool. Consumers (`BrowserLauncher`, Scrapling's `cdp_url: str = settings.BROWSER_CDP_URL` `vectai/core/discovery/dataclasses.py:106`, `ScraplingConfig.cdp_url`) all point at the same var.

---

## Phase 6 — Deep Dives

### 6.1 Topology: Compose profile vs K8s Deployment

| Layer | Docker Compose | Kubernetes |
|-------|----------------|------------|
| File | `docker-compose.production.yml:325` + volume `production_browserless_workspace:/workspace` | `k8s/base/deployment-browserless.yaml` `Deployment/browserless` + `Service/browserless:3001` |
| Activation | `docker compose --profile browserless up -d` (`browserless-separate-service.md` decision) | `kubectl apply -k k8s/base` (standalone — not merged with Django) |
| Scaling | `--scale browserless=5` (independent) | `spec.replicas: N` |
| DNS | `ws://browserless:3001` inside `vectai_network` | `ws://browserless.default.svc.cluster.local:3001` |
| Config | `.envs/.production/.browserless` → `BROWSER_CDP_URL` token in `.envs/.production/.django:106` | `Secret` for `TOKEN` + `ConfigMap` for `CONCURRENT/TIMEOUT/...` (drift fixed 2026-08 — see below) |
| Health | `curl -sf http://localhost:${PORT}/pressure?token=${TOKEN}` every 30 s | same liveness probe (add when promoting) |

K8s drift fixed 2026-08: `MAX_CONCURRENT_SESSIONS=10` → `CONCURRENT=5`, `CONNECTION_TIMEOUT 300000` → `TIMEOUT=120000`, `PREBOOT_CHROME=true` + `POOL_MIN=2`, `dns: [1.1.1.1,8.8.8.8]`. Old K8s used v1 var `MAX_CONCURRENT_SESSIONS` (ignored by v2 image; silently fell back to default 10, doubling memory vs Compose).

### 6.2 Browserless Grid (Chrome-as-a-Service)

Image `ghcr.io/browserless/chromium:v2.52.2` (`docker-compose.production.yml:326`). Runtime `build/config.js` maps legacy vars: `MAX_CONCURRENT_SESSIONS→CONCURRENT`, `CONNECTION_TIMEOUT→TIMEOUT`. Key tunables:

* `CONCURRENT=5` — hard slot ceiling. Each slot = one `Browser` (1 `ws://127.0.0.1:38763/devtools/browser/<id>`) plus optional contexts/pages.
* `QUEUED=10` `QUEUE_TIMEOUT=30000` — requests beyond `5` queue FIFO; caller gets `429` (discovery task sees `SoftTimeLimitExceeded` and returns error dict to chord).
* `TIMEOUT=120000` — session kill after 120 s of activity (was 300 s → Turnstile hangs 160–289 s held slots). Aligns with `discover_source_task` `soft 120 hard 180`.
* `PRE_BOOT_CHROME=true POOL_MIN=2` — avoids `browser:0` cold start before first `group(discover_source_task)` chord fans out.
* `HEALTH=true EXIT_ON_HEALTH_FAILURE=true MAX_CPU_PERCENT=85 MAX_MEMORY_PERCENT=85` — `GET /pressure` checks cgroup v2 CPU/memory; unhealthy → new requests `503`, container exits if health fails repeatedly. Verified healthy: `{"pressure":{"cpu":6,"memory":5,"running":0,"queued":0,"isAvailable":true,"maxConcurrent":5,"maxQueued":10}}`.
* `shm_size: 2gb` — Chromium `/dev/shm` for GPU/compositor buffers.
* `dns: [1.1.1.1, 8.8.8.8] dns_opt: timeout:2 attempts:2` — fixes `ERR_NAME_NOT_RESOLVED https://brunhild.challenges.cloudflare.com/...` (was using Docker internal resolver that dropped that host).

### 6.3 DualBrowserPoolManager — local fallback & concurrency

`vectai/browser_manager/services/dualbrowserpool.py:113` — singleton `worker_browsers`.

```python
async with worker_browsers.acquire(BrowserType.CHROME) as cdp_url:
    # pool: cdp_url = http://localhost:<free_port>
    # grid: cdp_url = ws://browserless:3001?token=...
    browser = await p.chromium.connect_over_cdp(cdp_url, slow_mo=...)
```

* `_get_pool_config()` reads `AUTOFILL_ENGINE` (`BROWSER_POOL_SIZE=4`, `BROWSER_POOL_CDP_BASE_PORT`, `BROWSER_ENABLE_LIGHTPANDA`).
* `ChromeSlot {cdp_port, process, in_use, user_data_dir}` per slot; `is_healthy` checks `returncode is None`.
* `asyncio.Semaphore(pool_size)` gates cross-engine concurrency (Chrome + Lightpanda). Lazy `initialize()` spawns Lightpanda (opt-in) then 4 chromes each with `tempfile.mkdtemp` `user_data_dir` isolation, flags `--remote-debugging-port`, `--disable-blink-features=AutomationControlled`, `--disable-dev-shm-usage`, `--headless=new` when `DEBUG=False`.
* `_wait_for_cdp` polls `/json/version` `15×1s` with `2s` timeout per attempt (see `browserless` health impl §6.9). `stop_all()` for Celery signal handlers: `terminate→kill` + `shutil.rmtree` + `clear()`.
* Grid fast-path `dualbrowserpool.py:185`:

  ```python
  if await _should_use_browserless():
      yield settings.BROWSER_CDP_URL; return
  ```

  So on healthy grid, `initialize()` never runs — no local chromes or Lightpanda. Only path that spawns processes. This is the correct fallback topology (not a second pool alongside the grid).

* Lightpanda opt-in: `BROWSER_ENABLE_LIGHTPANDA=False` default (`config/settings/base.py:1047`). `acquire(LIGHTPANDA)` falls back to Chrome with one `logger.info` (`_lightpanda_fallback_logged`). `BrowserLauncher` double-guards (`browser_launcher.py:103`) so Chrome gets full stealth instead of light `new_context()`.

### 6.4 BrowserLauncher — CDP + stealth + session hydration

`vectai/browser_manager/services/browser_launcher.py:78` — single place that does `connect_over_cdp`.

```python
if self.cdp_url:  # BYO grid or Browserless
    return await self._run_session(self.cdp_url, ...)
async with worker_browsers.acquire(self.browser_type) as cdp_url:
    return await self._run_session(cdp_url, ...)
```

`_run_session` (`line 85`):

1. `async_playwright()` + `p.chromium.connect_over_cdp(endpoint_url, slow_mo=AUTOFILL_BROWSER_SLOW_MO)` — same for grid or local slot.
2. Resolve `effective_type` (Lightpanda fallback).
3. Build `storage_kwargs` from `BrowserSession.storage_state` (encrypted). Injected via `BrowserSessionService(platform).load_storage_state(profile)` — one session per `(profile, platform)` (`unique_together`).
4. Create context: Lightpanda `browser.new_context(storage_state)` bare; Chrome builds `stealth_kwargs = {**ALL_EVASIONS_DISABLED_KWARGS, "navigator_webdriver": True, *_override=None}` then `Stealth(**).apply_stealth_async(context)` plus `_build_context_kwargs` from `browser_metadata` (viewport/user_agent/locale/timezone captured by extension).
5. `page.evaluate("navigator.webdriver")` debug log, then `DOMHelper(page)` callback (`strategy.fill_application` or `BrowserUseAgent.fill_and_submit`).
6. `finally` captures `await context.storage_state()` only when `storage_state is not None` and persists it via `refresh_session_storage_state` (never overwrites a good session with an empty run).

### 6.5 BYO Browser & CDP validation (SSRF guard)

`vectai/browser_manager/services/cdp_validator.py`

* `validate_cdp_url(url)` — enforces `ALLOWED_SCHEMES {ws,wss,http,https}`, rejects `BLOCKED_METADATA_HOSTNAMES {169.254.169.254, metadata.google.internal, ...}`, `*.internal/*.local/localhost`, then `getaddrinfo` every A record and rejects `is_loopback/is_private/is_link_local/is_multicast/is_reserved/is_unspecified`.
* `resolve_user_browser_cdp_url(user)` — `BrowserProvider.MANAGED→None` (use `settings.BROWSER_CDP_URL` grid or pool). Others map to `wss://connect.browserbase.com?apiKey=`, `wss://connect.steel.dev?apiKey=`, `wss://chrome.browserless.io?token=`, `Custom CDP` (validated). Called in `engine.py:399` and `tasks.py` → threaded into `BrowserLauncher(cdp_url=…)` so the user's runs leave the shared grid isolation.

### 6.6 Discovery scraping via Browserless (Scrapling AsyncStealthySession)

Discovery and browser automation converge on the same grid:

* `vectai/core/discovery/dataclasses.py:106` `ScraplingConfig.cdp_url: str = settings.BROWSER_CDP_URL` — plumbed through `ScraplingJobSource.build_config_from_run` → `AsyncStealthySession` (`scrapling_source.py:276`).
* `_stealth_session_kwargs()` sets `cdp_url`, `max_pages`, `headless`, `network_idle`, `solve_cloudflare=enable_anti_bot`, `block_webrtc`, `hide_canvas`, `google_search`, plus `cookies = _injected_cookies` from `BrowserSession` (if the board requires auth — e.g., LinkedIn). Cloudflare `cf_clearance/__cf_bm/_cfuvid` cookies are explicitly excluded (`scrapling_source.py:220`) — they're one-time challenges, not reusable auth.
* `vectai/discovery/engines.py:97` `JobSourceEngine._acquire_one` instantiates `JobSource.from_run` → `fetch(limit=…)` → `[asdict(ScrapedJob)]`. Drift-free: scraped dicts are normalized via `JobPersister`.
* Pipeline: `orchestrate_discovery` builds `group(discover_source_task.s(run_id, src) for src in sources) | aggregate | fan_out_embed | match`. Each `discover_source_task` is one slot on the grid for the scrape's lifetime.

### 6.7 Cloudflare / Turnstile hardening (2026-08 fix)

**Prod evidence before fix:** `docker logs vectai_production_browserless` tail showed `warn: Failed to execute 'postMessage' on 'DOMWindow' (challenges.cloudflare.com → null)` every 600–900 ms, `Turnstile Widget seem to have hung: zxc4u`, `403 https://www.ziprecruiter.com/fingerprint/observe` / `/ptid/` (tracker 403s), `brunhild.challenges.cloudflare.com ERR_NAME_NOT_RESOLVED`, sessions `160,289ms` & `289,204ms` killed by `SIGTERM received` deploy, not by timeout.

**Fixes applied:**

1. **`vectai/core/discovery/sources/scrapling_source.py:400`** — `is_blocked` allowlists tracker `403`s (`fingerprint/observe`, `/ptid`, `googleads`, `ccm/collect`) as `debug` not `warning`, pulls `429` out of `BLOCKED_STATUS_CODES` for distinct `Retry-After` logging, expands `200` challenge detection to `challenge-platform, cf-chl-widget, challenges.cloudflare.com/turnstile, turnstile widget seem to have hung`, adds Brunhild transient guard (return `False`).
2. **`retry_blocked_request` (`line 445`)** — if `sid==stealthy` already, returns `None` (drop, don't loop). Previously every block switched `default→stealthy` even when already on stealthy, looping 3× through Turnstile for 90 s.
3. **`SCRAPLING_MAX_BLOCKED_RETRIES` `3→2`** (`config/settings/base.py:1276` + `.envs/.production/.django:209`) — one stealthy retry then fail fast.
4. **Grid + task timeouts aligned** — grid `TIMEOUT 120s` + `QUEUE_TIMEOUT 30s`, celery `soft 120 hard 180`, grid `POOL_MIN 2` — hung boards free the slot in 2 min, not 5.
5. **DNS** — compose `dns: [1.1.1.1, 8.8.8.8]` for Brunhild host.

### 6.8 Task limits, timeouts & backpressure

| Task | Queue | `soft` / `hard` | `max_retries` | `rate_limit` | Purpose |
|------|-------|-----------------|---------------|--------------|---------|
| `discovery.discover_source` | `job_discovery` | `120` / `180` | `0` | `20/m` | One source scrape. Never Celery-retries (spider already did stealthy failover); chord callback `aggregate` always fires via `asdict(error=…)` dict. |
| `discovery.orchestrate` | `celery` | `60` / `90` | `2 backoff` | — | Lightweight chord builder. |
| `browser.apply_via_engine` | `browser_engine` | `480` / `600` | `1 delay 60` | `4/m` | Full hybrid pipeline (`HybridAutoFillEngine.execute`). |

`browser.apply_via_engine` is longer because it includes LLM runs (`BROWSER_USE_MAX_TOKENS 50000`, `max_failures 15`, `run_timeout 240` in `browser_use_agent.py`). `discover_source_task` rate `20/m` caps burst to grid `CONCURRENT+QUEUED = 15` without `429` hoarding. Failed discovery scrapes (`result.error`) still publish `source_metrics` + `progress` so RunOutcomePanel shows partial badge.

### 6.9 Health, logging & observability

**Health:** Grid exposes `GET /pressure?token=TOKEN` → `{"pressure":{"cpu":6,"memory":6,"running":0,"queued":0,"recentlyRejected":0,"isAvailable":true,"maxConcurrent":5,"maxQueued":10}}` polled `interval 30s timeout 10s retries 3 start_period 20s`. Grid code (`build/config.js`) self-checks cgroup v2 `cpu/memory` against `MAX_CPU_PERCENT/MAX_MEMORY_PERCENT 85` → `503` and with `EXIT_ON_HEALTH_FAILURE=true` exits on repeated unhealth.

**Pool health:** `DualBrowserPoolManager._is_browserless_healthy` hits `GET <base>/json/version` via `aiohttp` `2s` timeout (`dualbrowserpool.py:84`). Used as gate for grid vs pool fast-path. Does *not* carry token as query — works because internal `ws://browserless:3001?token=…` maps to `http://browserless:3001/json/version` (token ignored for that static JSON). `BROWSER_CDP_URL=ws://browserless:3001?token=b…` in-compose is internal net, not Traefik-exposed.

**Logging:** Browserless `v2.52.2` is `debug`-package based. Legacy `.envs` key `LOG_FORMAT=json` was correctly documented as “ignored today” (see that file's comment)—Docker `json-file` `max-size 20m *5` is the structured layer, scraped by `otel-collector` `receiver_creator/docker_observer` → Loki label `{service_name="browserless"}` (plus traces as child spans via W3C `traceparent` passed with every CDP `connect_over_cdp` from `BrowserLauncher`). Fix: `DEBUG=browserless* DEBUG_COLORS=false` yields clean timestamped lines (`2026-08-28T18:06:48.416Z browserless.io:hardware:debug ...`) without ANSI — ingestion was previously munged with `^[[34m` colors. Keep `LOG_FORMAT=json` for forward compat if browserless adds it.

**Metrics:** `GET /metrics?token=` and `/metrics/total` (daily), `GET /active` `/config` for ad-hoc. Grill on `browserless:3001/metrics` inside compose. Alert on `recentlyRejected>0` or `pressure.isAvailable==false`.

### 6.10 Security

* Token: `.envs/.production/.browserless:13` `TOKEN` (hex32 via `openssl rand -hex 32`) injected only as `?token=` on `BROWSER_CDP_URL` inside the Docker network (compose exposes `127.0.0.1:3001` only). Traefik route `browserless-secure-router` (`Host(browserless.localhost)`) is *not* used in prod; grid is not internet-exposed.
* SSRF: `validate_cdp_url` blocks metadata IP / internal / private after DNS resolution; tested for `custom_cdp` BYO path.
* Storage state: encrypted at rest via `django-encrypted-model-fields`; never logged.

---

## Failure Modes & Mitigations

| Failure | Symptom in logs | Mitigation now |
|---------|-----------------|----------------|
| Turnstile hang (ZipRecruiter, Indeed) | `challenge-platform` / `cf-chl-widget` in 200 body, `postMessage ... challenges.cloudflare.com` warnings | `is_blocked` catches 200 challenge → `stealthy` once (`MAX_BLOCKED_RETRIES=2`) → fail-fast `None` drop, task finishes with `error` dict, `aggregate` marks partial. Slot freed at `TIMEOUT 120s`. |
| Tracker 403 noise | `403 fingerprint/observe , /ptid , googleads` | allowlisted → `is_blocked=False`. No queue blow-up. |
| Brunhild DNS | `ERR_NAME_NOT_RESOLVED https://brunhild…` | compose `dns: 1.1.1.1/8.8.8.8` + `is_blocked` transient guard. |
| Grid down / container SIGTERM (deploy) | `SIGTERM received, saving and closing down` `Job has succeeded after 160289ms` | `BROWSER_CDP_FALLBACK_TO_LOCAL=True` → pool spawns local chromes (only when health fails) and finishes the `apply_via_engine` run. Deploys now have 2 min slot timeout so fewer in-flight sessions survive `SIGTERM`. |
| Grid at capacity | `recentlyRejected>0`, callers `429` | `QUEUED=10 QUEUE_TIMEOUT=30s` + `discover_source rate 20/m` → self-throttles; consider `--scale browserless=3` or separate discovery vs autofill grids if sustained. |
| Scraper burst (10 sources) | `running=5 queued=5` then `429` | `group` fans out but only 5 execute; `aggregate` handles partial failures via `error` field (not chord failure). |
| Local OOM (pool) | `Chrome CDP not ready after 15 attempts`, worker 6 Gi limit | Pool only spins when grid unhealthy (rare). 6 Gi worker limit covers `4×600MB + headroom`. Don't raise `BROWSER_POOL_SIZE>4` without raising grid `CONCURRENT`. |

---

## Operating Guide

```bash
# Full stack with grid (canonical prod)
docker compose --profile browserless up -d
# lightweight (no browser) — for CI
docker compose up -d
# tail
docker logs --tail 100 -f vectai_production_browserless
# pressure / metrics (inside host, token required)
curl -sf "http://localhost:3001/pressure?token=$TOKEN" | jq
curl -sf "http://localhost:3001/metrics?token=$TOKEN" | jq '.[].running'
# verify DNS fix
docker inspect vectai_production_browserless --format '{{.HostConfig.Dns}} {{.HostConfig.DnsOptions}}'
# discover_source limits live?
docker exec vectai-celeryworker_discovery-1 python -c "import django; django.setup(); from vectai.discovery.tasks.discover_source import discover_source_task; print(discover_source_task.soft_time_limit, discover_source_task.time_limit, discover_source_task.rate_limit)"
# redeploy after editing scrapling_source.py (host→container hot-patch for immediate, or rebuild)
docker compose build celeryworker_discovery && docker compose --profile browserless up -d
```

Frontend config: no env change needed — `BROWSER_CDP_URL` stays internal. BYO users set `Browser Provider = Browserless/Browserbase/Steel/Custom CDP + API key` in profile; `cdp_validator` proofs the routing.

---

## Cross-cutting Decisions & ADRs

* **Separate compose file (`-f docker-compose.browserless.yml`) mirrors K8s topology** (`browserless-separate-service.md`) — grid is optional infra, independently scalable, swapable (replace `browserless` with `steel` profile).
* **Single `BROWSER_CDP_URL` var** — discovery and autofill converge; no `SCRAPLING_CDP_URL` second var.
* **Timeout alignment 120/180** — 2026-08 post-mortem of 160–289 s Turnstile hangs decided to kill at 120 s grid + 180 s Celery hard, freeing slots 2.5× faster.
* **`MAX_BLOCKED_RETRIES 3→2` ADR** — one stealthy retry captures `solve_cloudflare` value; second loop is waste and never helped Brunhild transient.


<!-- **Related:** [Browserless Separate Compose](browserless-separate-service.md) · [Browser Automation Engine (Autofill)](../../docs/architecture/browser-automation-system-design.md) · [App Browser Manager](app_browser_manager.md) · [Celery Routing](celery_routing_architecture.md) · [Scrapling Integration](SCRAPLING-INTEGRATION.md) · [Docker Compose vs K8s](docker-compose-vs-kubernetes.md) -->
