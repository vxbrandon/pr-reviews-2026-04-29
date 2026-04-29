# PR #140 — Flow Inventory

Branch: `dev-coverage-and-odds-impro` → `dev`. Cited file/line numbers refer to the PR head.

## Summary

| | Count |
|---|---|
| New flows added | 14 |
| Modified flows | 9 |
| Removed flows | 0 |
| Total inventoried | 23 |

Most net-new logic is plumbed through (a) two new admin routers (`data_health`, expanded `pipeline_admin`), (b) four new background workers wired in `app.py` lifespan, (c) a new Kafka producer + consumer pair (`request_producer` → `request_analytics_service`), and (d) one extra Kafka producer (`lineup_producer`) driven by the lineup-scraper task. Existing odds/parlay routes are mostly modified only to (i) emit fire-and-forget tracking events and (ii) chase the `application/service/{odds,info,betting,calibration,lineup}/` package re-org.

---

## Flows

### F1: GET /admin/data-health/games/{game_id} — per-game completeness report
- Status: NEW
- Entry: `src/backend_odds/infrastructure/api/rest/routers/data_health.py:61`
- Pipeline:
  - route handler `get_game_completeness` `data_health.py:66`
  - → `_get_service()` builds `DataCompletenessService(get_session_factory())` `data_health.py:54`
  - → `service.check_game(game_id)` `application/service/info/data_completeness_service.py:70`
  - → SQL reads against `Games`, `Lineups`, `MatchStatistics`, `PlayerStatistics`, `Events` (whoscored/sofascore counts), `MigrationRunsAudit`
  - → `_evaluate_alerts(...)` `data_completeness_service.py:245`
- Side effects: read-only DB.
- Auth/observability: mounted under `/admin` with `Depends(verify_admin_token)` (`rest/routes.py:43`); flow tag `admin.operations` (`web/middleware/request_context.py:122`).

### F2: GET /admin/data-health/incomplete — list incomplete games in a window
- Status: NEW
- Entry: `data_health.py:84` `list_incomplete_games(season_id?, within_days)`
- Pipeline: `service.list_incomplete_games(...)` → loops `check_game` over upcoming/recent games → returns reports flagged with alerts.
- Side effects: read-only DB.
- Auth/observability: same admin guard; flow tag `admin.operations` (`request_context.py:123`).

### F3: POST /admin/migration/run-targeted (+ GET /admin/migration/{job_id})
- Status: NEW (added in this PR)
- Entry: `infrastructure/api/rest/routers/admin.py:446` (POST), `admin.py:490` (GET poll)
- Pipeline:
  - generate `job_id`, register in-memory `_migration_jobs` map
  - `BackgroundTasks.add_task(_run_targeted_migration_in_process, ...)`
  - background fn calls `scripts/migrations/async_migrate_whoscored_new_to_postgres_v2.py` and `scripts/migrations/async_migrate_sofascore_to_postgres_v2.py` for the supplied `game_ids`, then invalidates BO caches.
- Side effects: writes Postgres (`lineups`, `match_statistics`, etc.), invalidates Redis cache keys.
- Auth/observability: `verify_admin_token`; called by Backend-Server's prelive_lineups job per docstring `admin.py:457`.

### F4: GET/POST/DELETE /admin/config[/{name}][/ui] — runtime flag config
- Status: NEW (added in this PR)
- Entry: `admin.py:536` GET, `admin.py:546` POST, `admin.py:573` DELETE, `admin.py:593` GET ui
- Pipeline: list/set/clear flags via `runtime_flags` registry + Redis-backed override store; UI rendered with Jinja2.
- Side effects: writes/reads Redis runtime-flag keys.
- Auth/observability: every route guards on `verify_admin_token` (`admin.py:538,548,575,596`).

### F5: POST /admin/pipelines/{name}/refit — async refit + optional calibration
- Status: MODIFIED (calibrate_player_props branch added)
- Entry: `pipeline_admin.py:377`
- Pipeline: `_get_refit_manager()` → `RefitManager.start_refit(name, force, calibrate, calibrate_player_props)` (`application/service/calibration/refit_manager.py`).
- Side effects: spawns refit worker, writes model cache, writes calibration parameters to Redis/DB.
- Auth/observability: admin guard via `_admin_guard` mount.

### F6: POST /admin/pipelines/{name}/post-process — recompute statistics tables
- Status: NEW
- Entry: `pipeline_admin.py:436`
- Pipeline: `MatchStatisticsProcessor` → `PlayerStatisticsProcessor` (skip for shot-only configs) → `ZonalStatisticsProcessor` → `TeamZonalStatisticsProcessor` (each `pipeline_admin.py:485-498`).
- Side effects: bulk writes to `match_statistics`, `player_statistics`, `zonal_statistics`, `team_zonal_statistics`.

### F7: POST /admin/pipelines/{name}/calibrate-player-props — shortcut
- Status: NEW
- Entry: `pipeline_admin.py:508` — thin wrapper over `RefitManager.start_refit(force=False, calibrate_player_props=True)`.

### F8: GET /admin/sources/actions — static source→actions map
- Status: NEW
- Entry: `pipeline_admin.py:417` returns `SOURCE_ACTIONS` constant. Read-only.

### F9: POST /admin/migrate/{league_code} — canonical league migration
- Status: NEW
- Entry: `pipeline_admin.py:659`
- Pipeline:
  - validates `include_whoscored` / `include_sofascore` flags
  - → `migration_service.migrate_league(...)` `application/service/info/migration_service.py:89`
  - → `_run_whoscored` `migration_service.py:153` (invokes `scripts/migrations/async_migrate_whoscored_new_to_postgres_v2.py`)
  - → `_run_sofascore` `migration_service.py:212` (invokes `scripts/migrations/async_migrate_sofascore_to_postgres_v2.py`)
  - records both steps in the `migration_runs_audit` table (24h idempotency unless `force=True`).
- Side effects: writes Postgres (`lineups`, `match_statistics`, `events`, audit table), invalidates BO Redis cache.
- Auth/observability: admin guard.

### F10: POST /odds/get_parlay_odds — parlay pricing
- Status: MODIFIED
- Entry: `infrastructure/api/rest/routers/odds.py:101` `compute_parlay_odds`
- Pipeline: per-pick → `GetOddsForBet.get_odds_for_zap/zat/zao/team_prop` (`application/service/odds/odds_service.py`); combined product across legs.
- Net new in this PR: emits `OddsRequestEvent` per game_id via `odds_request_producer.emit(...)` `odds.py:181-194` (fire-and-forget).
- Side effects: Kafka produce on `odds.odds_requests` topic; Redis pricing cache via service layer.
- Auth/observability: route flow tag set in `request_context.py`.

### F11: POST /odds/get_parlay_expected_profit — EV calculation
- Status: MODIFIED (same instrumentation pattern)
- Entry: `odds.py:667` `calculate_parlay_expected_profit`
- Net new: emits `OddsRequestEvent` for every distinct `game_id` after computing `aggregate_bet_types` (`odds.py:837-855`).
- Side effects: Kafka produce.

### F12: POST /odds/get_player_odds, /get_zat_odds, /get_team_prop_odds, /get_zao_odds, /get_mom_odds (single-bet endpoints)
- Status: MODIFIED (only by package re-org imports — `application.service.odds.odds_service`, `application.service.betting.live_policy`)
- Entry: `odds.py:240, 319, 423, 475, 531`
- Pipeline: `GetOddsForBet.get_odds_for_<type>` → `MainOddsPipeline` → cache (`@flow_traced(Flow.ODDS_COMPUTE)` decorations on `odds_service.py:743,800,995,1050,1415`).

### F13: GET /info/games/market-odds — cached market consensus odds
- Status: MODIFIED (import path only)
- Entry: `infrastructure/api/rest/routers/info.py:354`
- Pipeline: lazy import of `application.service.odds.market_odds_service.MarketOddsService` (per-game cache 5 min, pipeline cache 15 min).
- Side effects: Redis read/write.
- Observability: `@flow_traced(Flow.DATA_INFO)` on `market_odds_service.py:154`.

### F14: WS — odds broadcast loop (existing)
- Status: MODIFIED (renamed import only)
- Entry: app startup `app.py:206` `asyncio.create_task(run_odds_broadcast(...))`
- Pipeline: subscribers driven by `infrastructure/api/websocket/manager.py`; runs every poll interval; uses `GetOddsForBet` (now `application.service.odds.odds_service`) to compute and push.
- Side effects: WS broadcast, Redis cache reads.

### F15: Kafka consumer — `odds.market_updates` (Kalshi/odds_api ingest)
- Status: MODIFIED — error/observability paths reworked (E.4, E.5)
- Entry: `infrastructure/kafka/consumer.py` `KafkaOddsConsumer` (started in `app.py` lifespan after `lineup_scraper_task`).
- Pipeline:
  - subscribe → `_consume_loop` → batch
  - per-message: `OddsUpdate.from_json_bytes()` (now narrow `_DESERIALIZE_ERRORS` tuple `consumer.py:87-92`; non-listed exceptions propagate)
  - `_parse_game_id(...)` instead of bare `int(key.match_id)` `consumer.py:430` — unparseable values increment `kafka.match_id.parse_error` OTEL counter and log once via `_log_bad_match_id_once` (deduped, cap 256).
  - unmapped `market_id` increments `kafka.odds_update.unknown_market` and emits one INFO via `_log_unknown_market_type_once` `consumer.py:502`.
- Side effects: Redis writes (`market:odds:{gid}`, `live_odds:{gid}` per existing logic), Postgres upserts, OTEL counters.
- Auth/observability: spans inside batch processing; counters defined `consumer.py:62-100`.

### F16: Kafka consumer — `odds.lineup_updates` (legacy / passive)
- Status: NEW (file added) but per its own docstring "no longer started by default" `infrastructure/kafka/lineup_consumer.py:1-13`.
- Entry: `KafkaLineupConsumer.start()` `lineup_consumer.py:105` — not invoked by `app.py`.
- Pipeline (if started): consume `odds.lineup_updates` → `LineupUpdate.from_json_bytes` → `_upsert_lineup` upsert into `lineups` table (`lineup_consumer.py:214`).
- Side effects: would write Postgres `lineups`, has manual `kafka.lineup.process_batch` span (`lineup_consumer.py:154`).
- Note: the active path is now F18 (in-task DB write + Kafka publish). Downstream services may still consume the topic.

### F17: Kafka producer — `odds.odds_requests` (analytics fan-out)
- Status: NEW
- Entry: `OddsRequestProducer` `infrastructure/kafka/request_producer.py:55`. Started in `app.py:294` (`odds_request_producer.start()`), stopped on shutdown `app.py:387`.
- Producers (call sites): F10 `odds.py:181-194`, F11 `odds.py:837-855`. Both wrapped in try/except — never fail the request.
- Side effects: Kafka produce to `odds.odds_requests` (key=`game_id`, value=JSON `OddsRequestEvent`).
- Observability: `tracer = trace.get_tracer("tbg.backend-odds.request-producer")` `request_producer.py:20`.

### F18: Background task — `run_lineup_scraper` (W4)
- Status: NEW
- Entry: `infrastructure/api/tasks.py:813`. Wired in `app.py:316` lifespan as `lineup_scraper_task`.
- Pipeline:
  - `games_repo.get_upcoming_games(season_ids, limit=100)` → adaptive frequency filter (>24h skip, 2-24h every 30 min, ≤2h or live every `LINEUP_SCRAPER_INTERVAL_SEC`/120s) `tasks.py:920-944`
  - per game → `scrape_lineup_from_db(session, game_id)` `infrastructure/workers/lineup_scraper.py:33` (returns `LineupUpdate` from current `lineups` rows, no external HTTP)
  - `LineupChangeDetector.detect(update)` `application/service/lineup/lineup_change_detector.py:66` (Redis-backed diff)
  - if changed → `_upsert_lineup_to_db` `tasks.py:828` (writes `LineupModel` upserts) + `KafkaLineupProducer.send(update)` `infrastructure/kafka/lineup_producer.py:66`
  - `producer.flush()` once per cycle if anything published.
- Side effects: Postgres `lineups`, Kafka `odds.lineup_updates`, Redis change-detector cache.
- Gate: `LINEUP_SCRAPER_ENABLED` env (default false); season list fed from `pipeline_configs.current_season_ids`.

### F19: Background task — `run_match_calibration_check` (W2)
- Status: NEW
- Entry: `tasks.py:1133`. Wired in `app.py:309` as `match_cal_task`.
- Pipeline:
  - `games_repo.get_upcoming_games(season_ids, limit=100)` `tasks.py:1183`
  - per game → `MatchCalibrationService.calibrate_match(game_id, home_team_id, away_team_id, trigger_type='market_odds')` `application/service/calibration/match_calibration_service.py:549` (cooldown gate, base lambdas from `team_pipeline.model`, `_load_market_moneyline_probs` from Redis, divergence check, then writes adjustment to Redis)
  - sleeps `MATCH_CALIBRATION_CHECK_INTERVAL_SEC` (default 120s)
- Side effects: Redis writes (per-match calibration adjustment, cooldown key), reads Redis market odds keys (`live_odds:{gid}` etc.), reads team-pipeline cache.
- Gate: `MATCH_CALIBRATION_ENABLED` env.

### F20: Background task — `run_scheduled_odds_fetch` (W3)
- Status: NEW
- Entry: `infrastructure/kafka/scheduled_odds_fetch.py:256`. Wired in `app.py:288` as `odds_fetch_task`.
- Pipeline (per cycle): if env keys present, `asyncio.to_thread`:
  - `_run_odds_api_fetch(market_categories)` → wraps `scripts/fetch_market_odds.py` (Odds API → Postgres) `scheduled_odds_fetch.py:130`
  - `_run_kalshi_fetch()` → wraps `scripts/fetch_kalshi_odds.py` (Postgres + Redis) `scheduled_odds_fetch.py:171`
  - `_run_player_prop_fetch()` → wraps `scripts/fetch_player_prop_odds.py` (Postgres + Redis) `scheduled_odds_fetch.py:210`
  - sleep `ODDS_FETCH_INTERVAL_SEC`
- Side effects: writes `external_odds.market_odds`, `external_odds.player_prop_odds`, Redis caches.
- Gate: `ODDS_FETCH_ENABLED` + presence of `ODDS_API_KEY` / `KALSHI_API_KEY`; `ODDS_FETCH_PLAYER_PROPS_ENABLED` for the third leg.

### F21: Background task — Request Analytics consumer (W1 part 2)
- Status: NEW
- Entry: `RequestAnalyticsService.start()` `application/service/info/request_analytics_service.py:68`. Wired in `app.py:298` (skipped when `--disable-kafka`).
- Pipeline:
  - background thread subscribes to `odds.odds_requests` `request_analytics_service.py:141`
  - per batch → `_aggregate_event(data)` `request_analytics_service.py:189` updates Redis: `req_analytics:match:{game_id}` hash (HINCRBY total_requests + per-market), HLL `req_analytics:match:{game_id}:users_hll`, sorted set `req_analytics:user_activity` (ZADD), with TTLs 24h/7d.
- Side effects: Redis-only writes; manual span `request_analytics.process_batch` `request_analytics_service.py:154`.

### F22: Background task — Exposure snapshot flush (W1 part 3)
- Status: NEW
- Entry: `_exposure_snapshot_loop` defined inline in `app.py:300`. Wired as `exposure_task` `app.py:325`.
- Pipeline: every 300s → `asyncio.to_thread(exposure_service.flush_snapshots)` `application/service/betting/exposure_service.py:52`.
- Side effects: reads Redis exposure keys, writes rows to `exposure_snapshots` Postgres table.

### F23: Startup hook — Alembic drift check
- Status: NEW
- Entry: `app.py:177-191` (top of `lifespan`)
- Pipeline: `check_alembic_drift()` `infrastructure/migrations/drift_check.py:130` compares each `MigrationTarget` (main alembic + alembic_ext_odds) head vs live DB `alembic_version`. On mismatch raises `AlembicDriftError` (re-raised in `lifespan` → blocks startup) when `ALEMBIC_DRIFT_FAIL=true`; otherwise logs.
- Side effects: read-only DB.

---

## Cross-cutting notes

- **Service package re-org** (no flow change): every router import was rewritten to `application.service.{odds,betting,info,calibration,lineup}.<service>`. Visible in diffs to `info.py:9`, `odds.py:14-15`, `tasks.py:13`, `app.py:7`. All are import-only.
- **Calibration service location**: `match_calibration_service.py` (1989 LOC) sits in `application/service/calibration/`; consumed only by F19 today, but exposes `MatchCalibrationService.calibrate_match` for future synchronous calls.
- **`MarketOddsApiService`** (`application/service/odds/market_odds_api_service.py:83`) is added but currently NOT wired to any router or task — only its DTOs are used (and `_dedup_player_props` referenced in tests). Treat as dead-code-on-arrival until a caller is added.
- **Lineup pipeline duplication**: `_upsert_lineup_to_db` is implemented inline in `tasks.py:828` AND in `KafkaLineupConsumer._upsert_lineup` (`lineup_consumer.py:214`). Same SQL pattern, two copies — flagged for the architecture pass.
- **Disable-kafka flag**: `--disable-kafka` cli flag now also gates F21 (RequestAnalyticsService) on top of the existing `KafkaOddsConsumer`. F17 (producer) still starts unconditionally because `OddsRequestProducer.start()` no-ops when `ODDS_REQUEST_TRACKING_ENABLED` is false.
