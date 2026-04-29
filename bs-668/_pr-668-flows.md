# Flows Specialist — Synthesis (Mid Level, Deep)

> Synthesized from 7 leaf reports under `pr-668-leaves/`:
> Foundation: [`flows-http-routes.md`](pr-668-leaves/flows-http-routes.md), [`flows-websocket.md`](pr-668-leaves/flows-websocket.md), [`flows-scheduled-kafka.md`](pr-668-leaves/flows-scheduled-kafka.md), [`flows-pipelines-workers.md`](pr-668-leaves/flows-pipelines-workers.md).
> Deep walks: [`flows-deep-sf-data.md`](pr-668-leaves/flows-deep-sf-data.md), [`flows-deep-admin-jobs.md`](pr-668-leaves/flows-deep-admin-jobs.md), [`flows-deep-cross-service.md`](pr-668-leaves/flows-deep-cross-service.md).

> **PRIORITY**: per user direction, this is the most thorough specialist branch in the review. Every flow walked end-to-end, every cross-service hop documented with `file:line`, header propagation matrix verified.

## TL;DR

PR #668 changes **45 HTTP routes** (40 new, 5 changed), adds **5 net-new scheduled jobs**, **2 Kafka emit sites**, **1 cross-service worker** (BS→BO targeted migration). **0 WebSocket flows** introduced or modified. The flow-correctness picture is mostly clean except for **3 specific gaps**: (a) 5 admin routes missing from `ROUTE_TO_FLOW`, (b) `request_id` propagation missing on the BS→BO admin call, (c) ContextVar loss in admin daemon threads. Auth posture on the BS→BO call is correct (JWT minted with `role=admin`).

## 1. Inventory by surface

```
Mobile/web client ─┬─ HTTP /info/sofascore/* (32 GET, public, no auth)
                   ├─ HTTP /info/get_games_in_date_range (existing route, internal rewrite)
                   └─ HTTP /admin/* (12 admin, 5 modified existing)

Admin client ─┬─ POST /admin/sofascore/{enrich,onboard}    + GET /admin/sofascore/status
              ├─ POST /admin/leagues/onboard{,*5 routes}
              ├─ POST /admin/scheduler/run/{job_name}      [body {dry_run}]
              └─ GET  /admin/scheduler/jobs

APScheduler (separate ECS task) ─┬─ live_fetch (5min)
                                  ├─ prelive_lineups (5min, was 10min)
                                  ├─ prelive_lineups_imminent (90s NEW, 1h window)
                                  ├─ postmatch_quick/full
                                  ├─ metadata_fetch
                                  └─ sf_enrich --tier {live|post|daily|weekly|all} (NEW)

OddsClient.get_parlay_odds      → Kafka odds.odds_requests   (OddsRequestEvent)
UserBetService (post-confirm)   → Kafka odds.user_bets       (BetPlacedEvent)

Cross-service:
  BS scheduler → BO /odds/admin/migration/run-targeted   (opt-in, JWT auth)
  BS web      → BO /live/is_live_batch                   (request path, ContextVar headers)
  BS web      → BO /get_parlay_odds, /live/is_live, etc. (existing, ContextVar headers)
```

WebSocket: **none added**. (Existing `WebSocketMetricsMiddleware` in `metrics_middleware.py` is unmodified; Stream Chat WS is third-party client-direct.)

## 2. End-to-end walks

### 2.1 Public reads — `/info/sofascore/*` × 32

[Full per-route detail in `flows-deep-sf-data.md`](pr-668-leaves/flows-deep-sf-data.md).

Common spine:

```
client GET /info/sofascore/{group}/{id}[/{field}]
   ▼ get_sofascore_data_service (DI)
SofascoreDataService.<method>
   ├─ id = await self.id_resolver.resolve_{match,player,team}(id)        ← UUID OR sf_*_id
   └─ return await self.repo.<method>(resolved_id, ...)
       ▼ AsyncSession query against sf_match_*, sf_player_*, sf_team_*, players, teams
       ▼ For *_image / *_media: S3 signed URL fallback to SofaScore CDN
```

Grouping:
- 17 single-match field routes (combined + 16 individual)
- 5 player routes (profile, national-team, recent-matches, image, media)
- 7 team routes (form, goal-distributions, stats, image, media, performance, squad)
- 3 league routes (matches, top-rated, image)
- 2 odds routes (match, league)
- 2 standings/top-players routes
- 2 ID-resolution routes

Observations across all 32:
- ✅ All 32 covered in `flow_context.ROUTE_TO_FLOW` as `data.sofascore`.
- ❌ Routes do NOT declare `response_model=` (schemas exist but unenforced).
- ❌ `SofascoreDataService` methods are NOT decorated with `@flow_traced(Flow.X)`.
- ✅ Public — no auth dep, no rate limit.

### 2.2 Admin job spawners — 4 routes

[Full detail in `flows-deep-admin-jobs.md`](pr-668-leaves/flows-deep-admin-jobs.md).

```
POST /admin/sofascore/enrich
  ▼ trigger_sf_enrich (admin/sofascore_routes.py:198-210)
  ├─ validate (league_code OR match_id)
  ├─ resolve pipeline-set from with_players/with_season flags
  ├─ _create_job → JobStatus.PENDING (in-process _jobs dict)
  └─ threading.Thread(_run_sf_enrich_sync, daemon=True).start()
        ▼ SfOrchestratorV4.run(...)   ← scrapes SF, writes sf_match_*, …
        └─ enrichment_audit.write_run(...)

POST /admin/sofascore/onboard
  ▼ trigger_sf_onboard
  └─ Thread(_run_sf_onboard_sync) — branches on action:
        ├─ STATUS:    onboarder.show_status()
        ├─ LINK:      onboarder.link_one(league_code) → UPDATE league_registry.sf_unique_tournament_id
        ├─ LINK_ALL:  onboarder.link_all() — iterate unlinked
        ├─ DISCOVER:  onboarder.discover(league_code, days_back) — read-only
        └─ CREATE:    onboarder.create(sf_id, name, short_code) — INSERT new row

POST /admin/leagues/onboard (+ list/get/delete/preset)
  ▼ services/admin/league_onboard_service.{start_onboard, get_status, list_onboard_jobs, cancel_onboard}
        ▼ Thread(_run_onboard_workflow) — multi-stage state machine ending in SfOrchestratorV4 invocation

GET /admin/sofascore/status
  ▼ get_sf_status — synchronous (no thread)
        ▼ JOIN league_registry × tournament_schedule × sf_match_details (counts)
```

### 2.3 Scheduler trigger with dry_run — `POST /admin/scheduler/run/{job_name}`

```
TriggerJobRequest{dry_run: bool}
  ▼ run_scheduler_job (scheduler_routes.py:84-180)
  ├─ all_jobs = {**SCRAPER_JOBS, **BACKEND_JOBS}
  ├─ 404 if job_name not present
  ├─ 400 if dry_run=true and not spec.supports_dry_run
  ├─ result = await spec.callable(dry_run=dry_run) | await spec.callable()
  └─ TriggerJobResponse{job_name, job_type, dry_run, result}
```

Job runs **inline on the request thread** (NOT in a separate thread). For long jobs this ties up a worker; for dry-runs the callable short-circuits before external effects.

For `live_fetch` wrapper (representative):
```
run_live_fetch(dry_run=False)
   ├─ args = parse([--max 100] + ([--dry-run] if dry_run else []))
   ├─ with job_context("live_fetch", args, enable_slack=not dry_run):
   │     await run_live_fetch_job(args, ctx)        ← orchestrator_v3
   └─ if not dry_run:
         await sync_lineup_updates_to_odds_service("live_fetch")     ← BS→BO post-step
```

### 2.4 `/info/get_games_in_date_range` — N+1 → batch

```
GameInfoService._compute_live_status_map(games, now)        ← game_info_service.py:117-181
   ├─ partition: terminal status     → False
   ├─ partition: LIVE_CHECK_MODE=ALWAYS → True
   ├─ partition: kickoff is None     → False
   ├─ partition: outside [-WINDOW_BEFORE, WINDOW_AFTER]      → False
   └─ remaining → odds_client.get_live_status_batch([…])
                       │  timeout: LIVE_STATUS_TIMEOUT (2s)
                       │  headers: get_propagation_headers() ✅ all 7 ContextVars
                       │
                       ├─ on success: dict[match_id → bool|None]
                       └─ on exception/None entry → time-based fallback (now > kickoff)
```

### 2.5 BS → BO targeted migration

[Full detail in `flows-deep-cross-service.md` §Y.](pr-668-leaves/flows-deep-cross-service.md)

```
Scheduler tick → run_live_fetch (or prelive_lineups, prelive_lineups_imminent)
   └─ if not dry_run:
         sync_lineup_updates_to_odds_service(job_name)           ← integrations/odds_service_client.py
              ├─ if not ODDS_SERVICE_SYNC_ENABLED: return
              ├─ wh_game_ids = await _fetch_recently_scraped_wh_game_ids(window=15min)
              └─ branch on ODDS_SERVICE_SYNC_MODE
                    ├─ "http"        → _post_targeted_migration(wh_game_ids)
                    │                       └─ POST {ODDS_SERVICE_URL}/odds/admin/migration/run-targeted
                    │                              body: TargetedMigrationRequest{game_ids, with_predicted_lineups, with_missing_players}
                    │                              headers: _admin_auth_headers()           ← ❌ ONLY Authorization
                    │                              timeout: 10s
                    │                              → 202 {job_id, …}
                    │
                    └─ "subprocess"  → conda run python <script_in_BO>     [dev/test only]
```

#### 2.5.1 Header propagation matrix — VERIFIED

| Caller | Authorization | request_id / flow / session | Verdict |
|---|---|---|---|
| `OddsClient.*` (request path) | n/a (in-VPC) | ✅ via `_propagation_headers()` → `get_propagation_headers()` | ✅ |
| `_post_targeted_migration` (scheduler path) | ✅ JWT (5min, `role=admin`, `sub=backend-server-scheduler`) | ❌ **NOT propagated** | ⚠️ |

`_admin_auth_headers()` at `integrations/odds_service_client.py:103-109` returns ONLY `Authorization`. Does NOT call `get_propagation_headers()`. Effect: BO logs the targeted-migration request without `X-Request-ID` correlation. Cross-service trace breaks at the scheduler→BO boundary. **One-line fix** documented in `flows-deep-cross-service.md` §Y.4.

### 2.6 Kafka producer — 2 emit sites

```
OddsClient.get_parlay_odds (existing route, post-response)
   └─ _emit_odds_request_event(endpoint="/get_parlay_odds", game_ids=[…], market_types=["parlay"])
            └─ producer.produce(topic="odds.odds_requests",
                                 key=user_id_ctx | "anonymous",
                                 value=OddsRequestEvent(...).to_json())

UserBetService (post bet-confirm)
   └─ self._emit_bet_placed_event(bet)
            └─ producer.produce(topic="odds.user_bets",
                                 key=user_id,
                                 value=BetPlacedEvent(...).to_json())
```

Both fire-and-forget (`producer.produce` never raises). ContextVars (`user_id_ctx`, `request_id_ctx`) are captured at emit time **on the request thread** — they propagate correctly into the event payload (verified in `OddsRequestEvent.session_id` ← `request_id_ctx`).

App lifecycle: `app.on_event("startup")` warms the singleton; `shutdown` flushes (5s timeout). Both wrapped in try/except.

### 2.7 Scheduled jobs — idempotency & locking

| Job | Idempotency | Lock | Notes |
|---|---|---|---|
| `live_fetch` (5 min) | ✅ ON CONFLICT DO UPDATE in BaseSink | SingleInstanceLock per job_id, in-process | post-step → BS→BO |
| `prelive_lineups` (5 min) | ✅ same | same | post-step → BS→BO |
| `prelive_lineups_imminent` (90 s NEW) | ✅ same | same; cross-instance NOT safe (process-local lock) | post-step → BS→BO; `enable_slack=False` |
| `postmatch_quick/full` | ✅ same | same | |
| `metadata_fetch` | ✅ same | same | |
| `sf_enrich` (per tier) | ✅ 24h gate at sf_enrich.py:268-291 (`status='skipped_idempotency'`) | same | |
| `dq_audit`, `bet_audit` | unknown | unknown | pre-existing FIXME(task-5/6) — broken imports |

Cross-instance scaling: if APScheduler runs in 2+ scheduler ECS tasks, the in-process lock does not coordinate. Both ticks fire, both POST to BO. Sinks tolerate duplicates (ON CONFLICT). BO must dedupe `game_ids` on the `/odds/admin/migration/run-targeted` side; **cross-repo question** for the BO team.

### 2.8 WebSocket — none

[`flows-websocket.md`](pr-668-leaves/flows-websocket.md) confirms zero WS routes added or modified. The existing `WebSocketMetricsMiddleware` and Stream Chat references in the codebase predate this PR. Cross-repo WS contract risk is **non-existent** for this PR.

## 3. Flow registry coverage

`flow_context.ROUTE_TO_FLOW` (`infrastructure/middleware/flow_context.py`):

| Routes | Mapped? | Flow value |
|---|---|---|
| `GET /info/sofascore/*` × 21 | ✅ | `data.sofascore` |
| `POST /admin/sofascore/{enrich,onboard}` × 2 | ✅ | `admin.scraper` |
| `GET /admin/sofascore/status` | ✅ | `admin.scraper` |
| `POST /admin/scheduler/run/{job_name}` | ✅ (existing) | `admin.operations` |
| `GET /admin/scheduler/jobs` | ✅ (existing) | `admin.operations` |
| `POST /admin/leagues/onboard` | **❌ MISSING** | — |
| `GET /admin/leagues/onboard` | **❌ MISSING** | — |
| `GET /admin/leagues/onboard/{job_id}` | **❌ MISSING** | — |
| `DELETE /admin/leagues/onboard/{job_id}` | **❌ MISSING** | — |
| `POST /admin/leagues/onboard/preset/{preset_name}` | **❌ MISSING** | — |

5 routes drift. Pre-commit hook `./scripts/check-flow-drift.sh` will fail.

## 4. Service-layer `@flow_traced` decoration

| Service | Decorated? |
|---|---|
| `services/sofascore/sofascore_data_service.py` | ❌ **none of ~22 methods** |
| `services/sofascore/sofascore_id_resolver.py` | ❌ |
| `services/admin/league_onboard_service.py` | ❌ |
| `services/admin/job_registry.py` | ❌ |
| `services/statistics/game_info_service.py` (`get_games_in_date_range` and `_compute_live_status_map`) | ❌ |
| `services/bets/user_bet_service.py` (existing methods) | ✅ `@flow_traced(Flow.BETTING_GENERATE)` at line 358 |
| `services/chat/chat_service.py` | ✅ multiple `@flow_traced` |

Drift: new code does not follow the project pattern. The middleware sets `flow_ctx` so log lines are tagged, but service-method-level OTEL spans are missing for these routes.

## 5. Cross-flow boundary matrix

| Source | Sink | Channel | Auth | request_id | flow | Verdict |
|---|---|---|---|---|---|---|
| Frontend | `/info/sofascore/*` × 32 | HTTPS GET | none | inbound from header | set by middleware | ✅ public |
| Admin client | `/admin/{sofascore,leagues/onboard,scheduler}/*` | HTTPS | `verify_admin_token` | ✅ | `admin.scraper` / `admin.operations` (5 missing) | ⚠️ 5 routes drift |
| BS web | BO `/get_parlay_odds`, `/live/is_live`, `/live/is_live_batch` | HTTPS | n/a | ✅ | ✅ | ✅ |
| BS scheduler | BO `/odds/admin/migration/run-targeted` | HTTPS | ✅ JWT | **❌ MISSING** | **❌ MISSING** | ⚠️ |
| BS web | Kafka `odds.odds_requests` | producer | n/a | ✅ via OddsRequestEvent.session_id | n/a | ✅ |
| BS web | Kafka `odds.user_bets` | producer | n/a | n/a in event schema | n/a | ✅ |
| BS app | SofaScore APIs | HTTPS via fetcher chain | n/a | n/a | n/a | ✅ |
| BS app | S3 (images, audit artifacts) | S3 SDK | IAM | n/a | n/a | ✅ |
| BS app | WebSocket (any direction) | — | — | — | — | **none** |

## 6. Cross-repo deploy ordering (flow correctness)

| Required before | Otherwise |
|---|---|
| BO `/live/is_live_batch` shipped | All `is_live` checks fall back to time-based silently — live-betting tile correctness regresses |
| BO `/odds/admin/migration/run-targeted` shipped | Post-scrape sync logs warn; BO stays stale on lineups (no client impact) |
| BO Kafka consumers shipped | Events queue in topic; topic retention bounds lag (KAFKA_BETTING_EVENTS_ENABLED defaults false) |

## 7. Top flow-correctness gaps (rolled up to risk)

1. **5 admin routes missing from `ROUTE_TO_FLOW`** (HIGH).
2. **`request_id` / `flow` not propagated on BS→BO admin call** (HIGH — one-line fix in `_admin_auth_headers`).
3. **ContextVar loss in admin daemon threads** (MED — wrap with `contextvars.copy_context()`).
4. **`@flow_traced` missing on new SF/admin services** (MED — service-method spans missing).
5. **Logging-formatter import drift in `src/scheduler/main.py:34`** (HIGH — scheduler may fail at startup).
6. **Cross-instance scheduler scale-out not safe for SingleInstanceLock** (LOW — process-local lock).
7. **BS does not dedupe `game_ids` across overlapping 90s ticks → 15min window** — relies on BO idempotency (MED — cross-repo question).

These propagate to the risk specialist's [`risk-flow-correctness.md`](pr-668-leaves/risk-flow-correctness.md) leaf.
