# PR #140 — Flow Inventory

Branch `dev-coverage-and-odds-impro` → `dev`. Synthesized from four child specialist reports:
- HTTP / REST → `outputs/_pr-140-flows-http.md`
- WebSocket → `outputs/_pr-140-flows-ws.md`
- Scheduled tasks + Kafka → `outputs/_pr-140-flows-tasks.md`
- Pipelines / service workers → `outputs/_pr-140-flows-pipelines.md`

A standalone unified-style first pass remains at `outputs/_pr-140-flows-v1.md`. The hierarchical version below supersedes it; v1 is kept only as a sanity cross-reference. All `file:line` citations refer to the PR-head tree; specialists verified each by reading head files (not raw diff).

---

## Summary

| Class | New | Modified | Removed | Total |
|---|---:|---:|---:|---:|
| A. HTTP / REST | 19 | 11 | 0 | 30 |
| B. WebSocket | 0 | 3 (import-only) | 0 | 3 (touched lines) |
| C. Scheduled tasks + Kafka | 6 tasks, 2 producers, 2 consumers | 1 consumer (`KafkaOddsConsumer`) | 0 | 11 |
| D. Pipelines / service workers | 8 | 6 | 0 | 14 |
| **Total flows inventoried** | **35 NEW, 21 MODIFIED, 0 REMOVED** | | | **~58** |

### Headline observability gaps

1. **17 admin/pipeline-admin routes drop into the `""` flow.** `ROUTE_TO_FLOW` keys for these paths lack the `/admin/` prefix the routers are mounted under (`rest/routes.py:40,42`); only the two new `/admin/data-health/*` entries actually resolve. Pre-existing wiring bug, but PR #140 adds 15 more orphans (see Class A and the v1 backup).
2. **No `@flow_traced` on any new admin / data-health / migration handler.** Service entries `DataCompletenessService.check_game`, `MigrationService.migrate_league`, `RefitManager.start_refit`, `MatchCalibrationService.calibrate_match`, `ExposureService.flush_snapshots` all ship without flow decorators. Only the existing odds endpoints + `MarketOddsService.get_market_odds` retain trace coverage.
3. **`KafkaLineupConsumer` is wired but not started** in `app.py` lifespan — file lives under `infrastructure/kafka/lineup_consumer.py:46`, lifespan never calls `.start()`. Producer side (`KafkaLineupProducer`) is live; consumer is dead code in this PR's runtime.
4. **`MarketOddsApiService`** (`application/service/odds/market_odds_api_service.py:83`, ~450 LOC plus DTOs) is added but unwired — no router, no scheduled task, no test imports. Treat as dead-code-on-arrival.
5. `idle_cleanup_task.cancel()` runs after `await ... gather(...)` and is never awaited (`infrastructure/web/app.py:397`) — minor coroutine leak on shutdown.
6. `_warm_market_odds_loop` has no env gate (`app.py:256-278`) — unconditional 240s Redis writes even in dev / disable-kafka mode.

### Cross-class re-org context

The bulk of the line-count delta in routers, `app.py`, `tasks.py`, and `manager.py:14` / `manager_typed.py:12` / `handler.py:26` is a single mechanical re-org: services moved from `application/service/<svc>.py` to `application/service/{odds,betting,info,calibration,lineup}/<svc>.py`. Where a flow is tagged "MODIFIED (import path only)" in a class table, that's the entire delta — no behavioural change, no decorator change. Real new behaviour clusters in (a) the new admin/data-health routers, (b) the four new lifespan tasks, (c) the calibration/cluster pipeline branches.

---

## Class A: HTTP / REST flows

Source: `outputs/_pr-140-flows-http.md`. Full router-by-router tables there; key entries reproduced inline.

### A.1 Summary table (new + non-trivially-modified routes only)

| # | Method + Path | Status | Router file:line | Service entry | flow_traced | ROUTE_TO_FLOW |
|---|---|---|---|---|---|---|
| F-HTTP-1 | POST `/auth/admin/login` | NEW | `rest/routers/auth.py:35` | `get_admin_password` + `create_access_token` | none | missing |
| F-HTTP-2 | POST `/auth/admin/logout` | NEW | `auth.py:74` | clears cookie | none | missing |
| F-HTTP-3 | GET `/admin/data-health/games/{game_id}` | NEW | `data_health.py:61` | `DataCompletenessService.check_game` (`info/data_completeness_service.py:70`) | none | present (`request_context.py:123` → `admin.operations`) |
| F-HTTP-4 | GET `/admin/data-health/incomplete` | NEW | `data_health.py:84` | `DataCompletenessService.list_incomplete_games` | none | present (`request_context.py:124`) |
| F-HTTP-5 | POST `/admin/cache/invalidate-lineup/{game_id}` | NEW | `admin.py:113` | `RepositoryFactory.lineups.clear_cache_for_game` | none | missing |
| F-HTTP-6 | POST `/admin/cache/invalidate-predicted-lineup/{game_id}` | NEW | `admin.py:142` | `predicted_lineups.clear_cache_for_game` | none | missing |
| F-HTTP-7 | POST `/admin/cache/invalidate-missing-players/{game_id}` | NEW | `admin.py:161` | `missing_players.clear_cache_for_game` | none | missing |
| F-HTTP-8 | POST `/admin/cache/invalidate-game/{game_id}` | NEW | `admin.py:176` | repo + Redis pattern delete | none | missing |
| F-HTTP-9 | GET `/admin/cache/sizes` | NEW | `admin.py:273` | repo `cache_size()` reads | none | missing |
| F-HTTP-10 | POST `/admin/migration/run-targeted` | NEW | `admin.py:446` | `BackgroundTasks.add_task(_run_targeted_migration_in_process)` (`admin.py:356`) → `scripts/migrations/async_migrate_whoscored_new_to_postgres_v2.py` | none | missing |
| F-HTTP-11 | GET `/admin/migration/{job_id}` | NEW | `admin.py:490` | in-memory `_migration_jobs` dict (`admin.py:338`) | none | missing |
| F-HTTP-12 | GET `/admin/config` | NEW | `admin.py:536` | `list_runtime_flags` | none | missing |
| F-HTTP-13 | POST `/admin/config/{name}` | NEW | `admin.py:546` | `set_runtime_flag` (Redis-backed) | none | missing |
| F-HTTP-14 | DELETE `/admin/config/{name}` | NEW | `admin.py:573` | `clear_runtime_flag` | none | missing |
| F-HTTP-15 | GET `/admin/config/ui` | NEW | `admin.py:593` | Jinja2 `admin_config.html` | none | missing |
| F-HTTP-16 | POST `/admin/pipelines/{name}/assign-and-fit` | NEW | `pipeline_admin.py:187` | `repo.add_season` + `RefitManager.start_refit` + saga rollback | none | missing |
| F-HTTP-17 | POST `/admin/pipelines/{name}/refit` | MODIFIED (`calibrate_player_props` query added) | `pipeline_admin.py:377` | `RefitManager.start_refit(name, force, calibrate, calibrate_player_props)` | none | partial (key lacks `/admin/` prefix) |
| F-HTTP-18 | POST `/admin/pipelines/{name}/post-process` | NEW | `pipeline_admin.py:436` | `MatchStatisticsProcessor`, `PlayerStatisticsProcessor`, `ZonalStatisticsProcessor`, `TeamZonalStatisticsProcessor` (sync, bulk DB) | none | missing |
| F-HTTP-19 | POST `/admin/pipelines/{name}/calibrate-player-props` | NEW | `pipeline_admin.py:508` | `RefitManager.start_refit(force=False, calibrate_player_props=True)` | none | partial (same prefix mismatch) |
| F-HTTP-20 | GET `/admin/sources/actions` | NEW | `pipeline_admin.py:417` | returns `SOURCE_ACTIONS` constant | none | missing |
| F-HTTP-21 | POST `/admin/migrate/{league_code}` | NEW | `pipeline_admin.py:659` | `MigrationService.migrate_league` (`info/migration_service.py:89`) → `_run_whoscored` (`:153`) → `_run_sofascore` (`:212`) | none | missing |
| F-HTTP-22 | POST `/get_parlay_odds` | MODIFIED (emits `OddsRequestEvent` per leg) | `odds.py:101`, emit at `:181-194` | `GetOddsForBet.get_odds_for_<type>` (`odds/odds_service.py:743/800/995/1050/1415`) + `odds_request_producer.emit` | service `@flow_traced(Flow.ODDS_COMPUTE)` | present |
| F-HTTP-23 | POST `/get_parlay_expected_profit` | MODIFIED (emits per game_id) | `odds.py:667`, emit at `:837-855` | same service entries + producer | service-level | present |
| F-HTTP-24..27 | POST `/get_player_odds`, `/get_zat_odds`, `/get_team_prop_odds`, `/get_zao_odds`, `/get_mom_odds` | MODIFIED (import path only) | `odds.py:240, 319, 423, 475, 531` | corresponding `GetOddsForBet.get_odds_for_*` `@flow_traced(Flow.ODDS_COMPUTE)` | service | present |
| F-HTTP-28 | POST `/builder/game/{game_id}/all-odds` | MODIFIED (emits `OddsRequestEvent("all_odds")`) | `builder.py:233`, emit at `:294-307` | `BuilderService.get_all_odds_for_game` + `IdMappingRepository` + producer | none on builder service | present (`odds.compute`) |
| F-HTTP-29 | GET `/info/games/market-odds` | MODIFIED (import only) | `info.py:354` | `MarketOddsService.get_market_odds` `@flow_traced(Flow.DATA_INFO)` (`odds/market_odds_service.py:154`) | service | present |
| F-HTTP-30 | `/live/*`, `/parameters/*`, remaining `/info/*` | MODIFIED (import-only) | various | renamed services under `betting/`, `calibration/`, `info/` | unchanged | mostly present (one pre-existing gap on `POST /live/is_live_batch` `live.py:75`) |

### A.2 Auth model

- New `auth.router` issues admin JWTs as HttpOnly cookies (F-HTTP-1) — public endpoint by design (it's the issuer).
- `admin.router`, `pipeline_admin.router`, `data_health.router` are mounted with `Depends(verify_admin_token)` shared dep (`rest/routes.py:40-46`). The runtime-flag block additionally re-asserts the dep per-handler (`admin.py:538,548,575,596`), which is belt-and-suspenders but not load-bearing.
- See risk specialist for the `secure=False` cookie semantics (intentional per docstring; TLS terminates upstream).

### A.3 Key HTTP flow walks

#### F-HTTP-10 / Targeted migration (BS → BO)
1. `POST /admin/migration/run-targeted` `admin.py:446` validates `game_ids` non-empty (`:465`)
2. mint `job_id`, register `_migration_jobs[job_id]` (`:468-477`)
3. `BackgroundTasks.add_task(_run_targeted_migration_in_process, ...)` (`:479`)
4. background fn dynamically loads `scripts/migrations/async_migrate_whoscored_new_to_postgres_v2.py` (`admin.py:381-392`) and runs `main(Namespace(...))`
5. on success → cache invalidation helpers (per docstring `:457`)
- Side effects: writes `lineups`, `predicted_lineup_players`, `missing_players`, `match_statistics`, …; Redis pattern-delete on `mom_odds:*`, `prob:zap:*`, `match_cal:*`
- Status: NEW; logs `flow=""`.

#### F-HTTP-21 / League migration facade
1. `POST /admin/migrate/{league_code}` `pipeline_admin.py:659` → `MigrationService.migrate_league(...)` (`info/migration_service.py:89`)
2. `_run_whoscored` (`migration_service.py:153`) — invokes WhoScored migrator script, audits in `migration_runs_audit`
3. `_run_sofascore` (`migration_service.py:212`) — gated on WhoScored success unless `force=True`
4. 24h idempotency via the audit table.
- Side effects: large Postgres writes, post-completion Redis cache invalidations.

#### F-HTTP-16 / Assign-and-fit saga
1. `POST /admin/pipelines/{name}/assign-and-fit` `pipeline_admin.py:187`
2. idempotency probe — `noop` when season already in group + last refit `swapped` (`:213-230`)
3. snapshot prior assignment (`_find_any_assignment` `:237`)
4. `repo.add_season(...)` (`:240-248`)
5. `RefitManager.start_refit(name, force, calibrate, calibrate_player_props)` (`:259`); 409 → `_rollback_assignment` (`:269`)
6. `_wait_for_refit_terminal(...)` poll until terminal status or 504 → rollback
7. on `failed` → rollback + structured error; on success → `_reload_routing_map()`.

#### F-HTTP-1 / Auth login
- `POST /auth/admin/login` `auth.py:35`: validate body password → `create_access_token({"sub":"admin","role":"admin"})` → `JSONResponse` + `Set-Cookie admin_token` (HttpOnly, samesite=lax, secure=False per docstring)
- Duplicates Backend-Server's `/auth/admin/login` so the same JWT works against either service.

---

## Class B: WebSocket flows

Source: `outputs/_pr-140-flows-ws.md`. Full subscription/push tables there.

### B.1 Summary

- New subscription channels: 0
- New server push types: 0
- Protocol-breaking changes: 1 docs-only example field rename (`zao` → `zao_bets` in `docs/api/websocket_subscription_api.md:363`); runtime payload built by `manager.py` is unchanged.
- PR-touched lines (3 import-only edits):
  - `infrastructure/api/websocket/manager.py:14` — `GetOddsForBet` → `application.service.odds.odds_service`
  - `infrastructure/api/websocket/manager_typed.py:12` — same rename, **dormant** (file isn't imported by `ws_dependencies`)
  - `infrastructure/api/websocket/handler.py:26` — `BuilderService` → `application.service.odds.builder_service`

### B.2 Existing subscription dispatch (unchanged in this PR; reproduced for context)

`WebSocketHandler.handle_message` (`handler.py:152-191`):

| Channel name | Handler | file:line | Service called |
|---|---|---|---|
| `subscribe` (zap/zat/zao/team_prop/mom) | `handle_subscribe` | `handler.py:213` | `ConnectionManager.update_subscription` `manager.py:518` → `GetOddsForBet` |
| `subscribe` (`is_builder` / `is_builder_leaders`) | `handle_builder_subscribe` | `handler.py:256` | `BuilderService.generate_quick_picks` |
| `unsubscribe` | `handle_unsubscribe` | `handler.py:314` | `ConnectionManager.remove_subscription` |
| `subscribe_spotlight` | `handle_subscribe_spotlight` | `handler.py:338` | `ConnectionManager.subscribe_spotlight` `manager.py:968` |
| `unsubscribe_spotlight` | `handle_unsubscribe_spotlight` | `handler.py:382` | `ConnectionManager.unsubscribe_spotlight` `manager.py:1035` |
| `ping` | `handle_ping` | `handler.py:327` | none |

13 server push types catalogued in the WS child report (`outputs/_pr-140-flows-ws.md` table starting at L34) — none modified.

### B.3 Key WS flow walks (illustrating service-rename impact)

#### F-WS-1 / Game odds subscribe
1. Client → `/ws/{client_id}` `ws_routes.py:14`
2. `ConnectionManager.connect` `manager.py:70` (span `websocket.connect`)
3. Client sends `{action:"subscribe", game_id, bet_type, ...}`
4. `WebSocketHandler.handle_message` dispatches `handler.py:166`
5. `handle_subscribe` parses `SubscribeRequest`; if `is_builder*` → F-WS-2 (`handler.py:227`)
6. `manager.update_subscription(...)` `manager.py:518` — invokes `GetOddsForBet` (now under `application.service.odds.odds_service`)
7. Initial odds pushed as `odds_update` `manager.py:416`; ongoing updates via the `run_odds_broadcast` loop (see F-TASK-A in Class C).

#### F-WS-2 / Builder subscribe
1. Same dispatch as F-WS-1 step 4
2. `handle_builder_subscribe` `handler.py:256` validates `zone_group` (`CUSTOM_ZONE_GROUP` with `zones[]` or in `DEFAULT_ZONE_GROUP_CONFIGS`) `handler.py:265-280`
3. UUID `game_id` mapped via `manager.id_mapping_repo.map_game_id` `handler.py:285-286`
4. `BuilderService(self.manager.odd_service).generate_quick_picks(...)` in a thread `handler.py:299` — service now resolved from `application.service.odds.builder_service`
5. `manager.send_builder_odds_update(...)` emits `data={builder_quick_picks: ...}` `handler.py:303-307`.

### B.4 Spans (existing; PR adds none)

`websocket.connection` `handler.py:84`; `websocket.process_message` `handler.py:157`; `@traced` decorators on `handler.py:213/256/314/327/338/382`; manager spans `manager.py:70/100/166/518/920`.

---

## Class C: Scheduled tasks + Kafka

Source: `outputs/_pr-140-flows-tasks.md`. Full producer/consumer tables there.

### C.1 Lifespan-registered tasks

| # | Name | Trigger / interval | Wired at | Impl | Env-gate | Idempotent? |
|---|---|---|---|---|---|---|
| F-TASK-A | `run_odds_broadcast` | loop, 60s | `app.py:218` | `tasks.py:41` | always on | yes (read-only push) |
| F-TASK-B | `run_live_odds_broadcast` | loop, `LIVE_BETTING_POLICY.broadcast_interval_seconds` (10s default) | `app.py:221` | `tasks.py:97` | always on | yes |
| F-TASK-C | `run_spotlight_broadcast` | loop, 120s | `app.py:230` | `tasks.py:232` | always on | yes |
| F-TASK-D | `run_idle_cleanup` | loop, 300s | `app.py:237` | `tasks.py:400` | always on | yes |
| F-TASK-E | `run_expected_profit_task` | loop, 5m | `app.py:240` | `tasks.py:451` | always on | yes (Redis SET ex) |
| F-TASK-F | `run_scheduled_calibration` | loop, `CALIBRATION_INTERVAL_HOURS` (24h; sleeps first) | `app.py:243-244` | `tasks.py:1041` | `CALIBRATION_ENABLED=true` (`tasks.py:797`) | yes (`start_refit` returns conflict on overlap) |
| F-TASK-G | `_warm_market_odds_loop` | loop, 240s | `app.py:256-278` | inline coroutine | always on (no gate) | yes (cache overwrite) |
| **F-TASK-H** | **`run_scheduled_odds_fetch` (W3)** | loop, `ODDS_FETCH_INTERVAL_SEC` (300s) | `app.py:292` | `kafka/scheduled_odds_fetch.py:256` | `ODDS_FETCH_ENABLED=true` (`scheduled_odds_fetch.py:71`) | partial (DB upserts safe; sequential `await` chain ensures no overlap inside this task) |
| **F-TASK-I** | **`run_match_calibration_check` (W2)** | loop, `MATCH_CALIBRATION_CHECK_INTERVAL_SEC` (120s) | `app.py:305` | `tasks.py:1133` | `MATCH_CALIBRATION_ENABLED` | yes (cooldown + divergence check inside service) |
| **F-TASK-J** | **`run_lineup_scraper` (W4)** | loop, `LINEUP_SCRAPER_INTERVAL_SEC` (120s base) + adaptive (skip >24h, 30 min @ 2-24h, base @ <2h/live) | `app.py:329` | `tasks.py:813` | `LINEUP_SCRAPER_ENABLED` (`tasks.py:808`) | yes (`LineupChangeDetector` short-circuits unchanged; upsert by `(game_id, player_id)`) |
| **F-TASK-K** | **`_exposure_snapshot_loop`** | loop, sleep-first 300s | `app.py:349-366` | inline + `ExposureService.flush_snapshots` `exposure_service.py:143` | always on | row-level upsert per `(match, snapshot_at)` |

(F-TASK-H/I/J/K are the four net-new lifespan workers in this PR. F-TASK-A..F-TASK-G are existing tasks reproduced for context; F-TASK-G is also flagged for missing env-gate.)

Pre-existing `run_parameter_backup_task` (`tasks.py:341`) is defined but **not started** in current `app.py` lifespan; the Redis pub/sub `redis_subscriber_task` (`tasks.py:145`) call is commented out at `app.py:234`. Both pre-PR.

Lifespan teardown cancels the 10 active tasks (`app.py:372-374`) but `idle_cleanup_task.cancel()` runs after `gather` and is never awaited (`app.py:397`) — minor coroutine leak.

### C.2 Kafka producers

| Topic | Producer | Trigger | Schema | Config |
|---|---|---|---|---|
| `odds.lineup_updates` (`KAFKA_LINEUP_TOPIC`) | `KafkaLineupProducer.send` `kafka/lineup_producer.py:85,101`; call sites `tasks.py:1008,1021` | each F-TASK-J cycle that detects a change | `LineupUpdate` JSON; key `lineup:{game_id}` | `enable.idempotence=true, acks=all, lz4` (`lineup_producer.py:23`) |
| `odds.odds_requests` (`ODDS_REQUEST_KAFKA_TOPIC`) | `odds_request_producer.emit(OddsRequestEvent(...))` `kafka/request_producer.py:128`; call sites `rest/routers/odds.py:186,848`, `rest/routers/builder.py:297` | inline on REST odds-fetch endpoints (F-HTTP-22/23/28); fire-and-forget | `OddsRequestEvent(user_id, game_id, market_types, bet_types, ts_ms, session_id, device_id)` `request_producer.py:27` | `acks=1, linger.ms=50` (analytics-grade) |
| `odds.market_updates` (preexisting `KAFKA_ODDS_TOPIC`) | unchanged producer side in this PR | upstream (tbg-streaming) | `OddsUpdate` | n/a in this PR |

### C.3 Kafka consumers

| Topic | Consumer file:line | Started? | Handler | Side effects | Manual span |
|---|---|---|---|---|---|
| `odds.market_updates` | `kafka/consumer.py:228` (`KafkaOddsConsumer`) | yes — `app.py:316` unless `--disable-kafka` | `_process_batch` parses `OddsUpdate`, splits moneyline/totals/spreads, writes Redis `live_odds:{gid}` then upserts `MoneylineOdds/TotalsOdds/SpreadOdds` | Redis (TTL 30s live / 300s prematch) + DB upsert; sync offset commit | `kafka.process_batch`, `kafka.redis_write`, `kafka.db_upsert` (`consumer.py:379, 513, 519`); 3 OTEL counters (`kafka.match_id.parse_error`, `kafka.odds_update.deserialize_error`, `kafka.odds_update.unknown_market`) (`consumer.py:70-97`) |
| `odds.odds_requests` | `application/service/info/request_analytics_service.py:54` | yes — `app.py:344` unless `--disable-kafka` | `_aggregate_event` → Redis HINCRBY `req_analytics:match:{gid}` (TTL 86 400), HLL `pfadd` for unique users, ZADD `req_analytics:user_activity` (TTL 7d) | Redis only; sync offset commit | `request_analytics.process_batch` (`request_analytics_service.py:154`) |
| `odds.lineup_updates` | `kafka/lineup_consumer.py:46` (`KafkaLineupConsumer`) | **no** — module retained, lifespan never calls `.start()`; docstring at `lineup_consumer.py:1-12` confirms scraper now writes DB directly | would upsert `LineupModel` rows | n/a (not running) | defined `kafka.lineup.process_batch` `lineup_consumer.py:154` |

Net for `KafkaOddsConsumer` modifications: `BATCH_SIZE 2000→200`; narrow-except split between `OddsUpdate.from_json_bytes` deserialize (`_DESERIALIZE_ERRORS = (JSONDecodeError, UnicodeDecodeError, TypeError, ValueError)` at `consumer.py:87-92`, real bugs propagate) and `_parse_game_id` for non-integer `match_id`; bounded dedup sets (cap 256) for bad match_id, bad payload, unknown market_id surface counter spikes + one log per novel value.

### C.4 Key task flow walks

#### F-TASK-J / Lineup Scraper (W4)
- Trigger: `asyncio.create_task(run_lineup_scraper(...))` `app.py:329`
- Steps:
  - `games_repo.get_upcoming_games(season_ids, limit=100)` `tasks.py:915`
  - filter by lookahead window + per-game elapsed-since-last-scrape `tasks.py:923-957`
  - `scrape_lineup_from_db(session, game_id)` via `asyncio.to_thread` `tasks.py:975-979`
  - `LineupChangeDetector.detect(...)` (skip if unchanged) `tasks.py:988`
  - `_upsert_lineup_to_db` direct ORM upsert `tasks.py:850-888, 997` (duplicate of `KafkaLineupConsumer._upsert_lineup` `lineup_consumer.py:214` — flagged for arch pass)
  - `KafkaLineupProducer.send` then `flush()` per cycle `tasks.py:1008, 1021`
- Side effects: writes `lineups`; publishes Kafka.
- Failure mode: per-game try/except logs and continues; `producer.close()` on cancel; no DLQ.

#### F-TASK-I / Match calibration check (W2)
- Trigger: `app.py:305`. Loop every 120s.
- Steps: `games_repo.get_upcoming_games(...)` → per game `cal_service.calibrate_match(game_id, home, away, trigger_type="market_odds")` via `asyncio.to_thread` `tasks.py:1188`. See P-CAL in Class D.
- Failure mode: outer `try/except` logs sweep error and continues; cooldown enforced inside the service.

#### F-TASK-H / Scheduled external odds fetch (W3)
- Trigger: `app.py:292`; loop `ODDS_FETCH_INTERVAL_SEC` (300s) `scheduled_odds_fetch.py:309`.
- Steps: sequential `_run_odds_api_fetch(market_categories)` → `_run_kalshi_fetch()` → `_run_player_prop_fetch()` each via `asyncio.to_thread`.
- Side effects: DB writes (odds tables); Redis `live_odds:{gid}` (Kalshi) and `player_props:{gid}` (player props). Mutates `sys.path` once via `_ensure_script_imports` `scheduled_odds_fetch.py:97`.
- Failure mode: per-stage `try/except logger.exception`; no quota-exhaustion alerting.
- Idempotency note: `ODDS_FETCH_PLAYER_PROPS_MAX_EVENTS=10` per league caps quota; consecutive cycles always re-hit upstream. Sequential `await` chain ensures no overlap today; would break if moved to threadpool fan-out.

#### F-TASK-K / Exposure snapshot flush
- Trigger: `_exposure_snapshot_loop` `app.py:349`. Sleeps 300s **before** first run.
- Steps: `ExposureService.flush_snapshots()` via `asyncio.to_thread`; `SCAN_ITER match=exposure:match:*` (cluster-safe per-node); per match `HGETALL` → `(bet_count, total_stake, single_stake, parlay_stake)`; bulk INSERT into `ExposureSnapshots` (var95/var99 placeholder 0.0).
- Failure mode: `try/except logger.warning`; per-cycle continues.

#### F-TASK-L / `odds.odds_requests` producer/consumer pair (W1)
- Producer `app.py:339`; analytics consumer `app.py:343-344` unless `--disable-kafka`.
- Producer drops on `BufferError`/`Exception` with WARN log `request_producer.py:146-149`; consumer offsets committed sync, errors logged + 1s sleep `request_analytics_service.py:159-161`. Gated by `ODDS_REQUEST_TRACKING_ENABLED=true` (`request_producer.py:69`) — note this is a **separate** flag from `--disable-kafka`, so the producer can be live while the consumer is down (and vice-versa).

### C.5 Other findings

- v1 report's "ralph loop wiring" claim is **unsupported** — the only ralph artifacts in this PR are markdown docs (`docs/pr/ralph-loop-agents-guide.md`, `AI_DOCS/reference/2026-04-18_ralph-audit-fix-report.md`). No scripts, no scheduled run.
- `run_scheduled_calibration` polls `refit_manager.get_status` every 30s up to 1800s with no exponential backoff — fine, just notable.
- `run_scheduled_odds_fetch` ignores all repo arguments (`scheduled_odds_fetch.py:271-276`) — interface noise to clean up.

---

## Class D: Pipelines / service workers (the meat)

Source: `outputs/_pr-140-flows-pipelines.md`. The pipeline child has the deepest content — Class D below preserves its structure with light editing.

### D.1 Summary

- New service flows: 8 (P-CAL, P-PLAYER-CAL, P-CLUSTER, P-MARKET, P-REFIT, P-EXP, P-LINEUP, P-SIM-CAL)
- Modified pipeline branches: 6 (P-FIT, P-ZAP, P-ZAP-BATCH, P-PARLAY, P-TEAMPROP/ZAT, P-LIVE)
- New `@traced` spans: 1 (`simulation.simulate_match_calibrated` `simulation.py:43`)
- Cross-cutting helpers added/centralised:
  - `PricingContext` (`pricing_context.py`) — snapshots adj exactly once per (pipeline, game), threaded through all three ZAP paths + parlay; closes URA-T17/T19 staleness class.
  - `cache_fingerprint.py` (`team_lambda_fingerprint`, `player_prob_fingerprint`, `full_adj_fingerprint`) — mixes only the adj slice each cache reads, keeping hit rate while preventing mid-window staleness.
  - `vig_policy.apply_power_vig` — single source for power-vig math (consumed by `MarketOddsService`, `GetOddsForBet`, `MainPredictionPipeline._apply_vig_odds_power`); deduplicates three prior copies.

### D.2 Pipeline flow walks

#### P-FIT — pipeline-startup fit + market calibration (MODIFIED)
- Trigger: DI `ws_dependencies.py:286` per group; also from P-REFIT.
- `MainPredictionPipeline.fit_models` `main_pipeline.py:725` (`@traced pipeline.fit_models`)
  → `player_pipeline.fit_model` `player_action_pipeline.py:385`
  → `team_pipeline.fit_model` `team_model_pipeline.py:386`.
- If `use_market_calibration`: ordered loop over `calibration_order=[h2h, totals, spreads]` calling `team_pipeline.calibrate_with_market_odds / _totals_odds / _spreads_odds` `:768-1020`; per-game errors > `max_game_calibration_error` populate `_rejected_game_ids` (consumed at request time).
- NEW: `_fit_cluster_model` `:1471` and `_fit_lineup_model` `:1396` (Redis-cached). NEW: `calibrate_player_props` `:1118` (binomial-error gradient against player model).

#### P-ZAP — single-leg ZAP probability (MODIFIED)
- Trigger: F-HTTP-24 → `pipeline.get_prob_for_zap` `main_pipeline.py:2024` (`@traced pipeline.get_prob_for_zap`).
- (1) Redis GET `pricing_block:{gid}:{pid}:{action}` `:2065-89` (extreme-policy sentinel; if set return `None`).
- (2) `PricingContext(self, gid)` `:2119` + `_ensure_adj()` once → loads `MatchCalibrationAdjustment` from `match_cal_adj:{gid}` via `_get_match_calibration` `:237` (TTL-cached).
- (3) `player_prob_fingerprint(adj, pid, action)` mixed into key (URA-T19).
- (4) GET `prob:zap:...:adj=<fp>` (TTL ≈ 15 min).
- (5) Miss → `_sample_player_counts` `:2252`: team_totals from P-SIM-CAL × `player_pipeline.model.predict` × `zonal_prob` × cluster blend × `lineup_weight = expected_minutes / 90`.
- (6) `rng.binomial(n=team_totals, p=effective_rate)` vs line.

#### P-ZAP-BATCH — builder all-ZAP (MODIFIED)
- BuilderService → `pipeline.get_all_zap_odds_for_game` `batch_zap.py:690` (`@traced batch_zap.get_all_zap_odds_for_game`).
- Cache key `builder:all_zap:...:adj={full_adj_fingerprint}` `:719` (URA-T33).
- `_ensure_models_loaded` gate `:730` (URA-T30: returns `[]` + WARN if unfit, vs prior silent simulate).
- Single `simulate_match_calibrated(gid, n_simulations=3000, actions=all)` `:770` (was `simulate_match`, URA-T17), vectorized eval over `(player, action, line)`.

#### P-PARLAY — parlay leg pricing (MODIFIED)
- Trigger: F-HTTP-22 / F-HTTP-23 → `GetOddsForBet.get_odds_for_parlay` `odds_service.py:1416` → `_get_pipeline_parlay_odds` `:1474` → `pipeline.get_odds_for_parlay_bet` `parlay_odds.py:1592` (`@traced parlay.get_odds_for_parlay_bet`).
- `parlay_validator.validate(game_ids)` `live_policy.py:383` → reject ended → split live vs pre-match `odds_service.py:1430-50`.
- Live → `_get_live_parlay_odds` (matches picks against `live_odds:{gid}` snapshot).
- Pre-match: build one `PricingContext` per game_id `parlay_odds.py:1634-55` (URA-T4, every leg shares one adj snapshot), embed `full_adj_fingerprint` per game in cache key `:1664` (TTL 30s). Per game `_get_odds_for_match_parlay(bets, gid, ctx=...)` calls `simulate_match_calibrated` (was `simulate_match`) and `PricingContext.get_player_rate` for ZAP legs.
- Cross-match product clamped at `max_odds_multiplier`.

#### P-SIM-CAL — calibrated match simulation (NEW)
- Used by P-ZAP, P-ZAP-BATCH, P-PARLAY, P-TEAMPROP/ZAT.
- `SimulationEngine.simulate_match_calibrated` `simulation.py:43`.
- Reads adj via `pipeline._get_match_calibration` → `team_lambda_fingerprint(adj)` → key `odds:sim:calibrated:{gid}:{actions}:adj={fp}_v2`.
- Per-action confidence-blended lambdas `λ_final = c·factor + (1−c)·1.0` (`_apply_match_cal_lambda` `main_pipeline.py:270`) feed `simulate_team_actions_with_lambdas` (vine copula sampling from calibrated marginals — pre-PR fed raw lambdas and post-hoc-multiplied).
- Cache JSON `CacheTTL.SIMULATION` ≈ 5 min. Distinct namespace from `simulate_match`.

#### P-CAL — per-match calibration sweep (NEW)
- Trigger: F-TASK-I (`run_match_calibration_check` `tasks.py:1133`, every 120s).
- `MatchCalibrationService.calibrate_match(gid, home, away, trigger_type)` `match_calibration_service.py:549`.
- Gate on `is_match_calibration_enabled` + `_is_on_cooldown`. Resolve pipeline via injected router.
- Base lambdas `team_pipeline.model.predict_team_mean(..., "goals")`.
- `_load_market_moneyline_probs` from Redis `live_odds:{gid}` `:892` (URA-T31: missing key now degrades to identity factors instead of short-circuit).
- Compute `_divergence(model_probs, market_probs)`; if ≥ `MATCH_CALIBRATION_DIVERGENCE_THRESHOLD` run `_optimise_factors` `:813` (scalar sq-error min); confidence = `max(floor, 1 − residual/divergence)`.
- Player props (P-PLAYER-CAL) and `_calibrate_team_props` `:1617` (corners/cards μ optimisation) run unconditionally (URA-T27).
- URA-T27 guard `:701`: if moneyline agrees AND no player adj AND no team adj → `None`. Else build `MatchCalibrationAdjustment(home/away_lambda_factor, confidence, player_adjustments[str_pid][action]=factor, team_adjustments)`; SET `match_cal_adj:{gid}` (TTL ~30 min) + cooldown.
- Optional `pricing_block:*` sentinels via `_apply_extreme_policy` `:294`.

#### P-PLAYER-CAL — per-player prop calibration (NEW)
- `_calibrate_player_props` `match_calibration_service.py:1119`.
- `_load_player_props_from_redis(gid)` `:1512` (URA-T21: Redis-only, never DB).
- `resolve_unmatched_odds` maps API names → internal `player_id`.
- Per (player, action): `_get_fitted_r` `:966`, devig market via `devig_power`, infer implied rate, derive multiplicative factor; clamp `[0.3, 3.0]`; warn outside `[0.4, 2.5]` with structured `(pid, action, factor)`.
- Returns `{str(pid): {action: factor}}`.

#### P-TEAMPROP / P-ZAT (MODIFIED)
- F-HTTP-26 → `pipeline.get_odds_for_team_props` `main_pipeline.py:3323` and `pipeline.get_odds_for_zat` `:3373`.
- `get_probs_for_team_props` `:3104` cached at `team_props:{gid}:{zones}:unified_vig`.
- NEW analytical fast path `_get_team_props_analytical` `:3038` for full-pitch — Poisson outer-product after `_apply_match_cal_lambda(gid, λ, is_home)` `:3061-62` (~0.1ms vs 374ms sim). Apply `_apply_vig_odds_power`, filter by min/max odds.
- ZAT: `_get_team_action_rate × zonal_pipeline.get_combined_zonal_probs`; goals action gets `_apply_match_cal_lambda` `:3422-24` before zonal product.

#### P-LIVE — live odds read (MODIFIED)
- F-HTTP-24..27 when `_is_live_priceable` → `_get_live_odds_from_redis` `odds_service.py:449`.
- Simulate-mode override returns `_get_cached_simulated_odds` `:460-64`.
- GET `live_odds:{gid}`; missing → `NO_COVERAGE` vs `PENDING` via `_has_historical_odds` (DB).
- Reject if `age > LIVE_BETTING_POLICY.freshness_threshold_seconds` → `STALE`.
- `LiveOddsService.get_team_props` `live_odds_service.py:91` reformats moneyline/totals/spreads with power-vig (hold/gamma from pipeline config).

#### P-MARKET — batch market-odds blend (NEW)
- F-HTTP-29 → `MarketOddsService.get_market_odds_batch(game_ids)` `market_odds_service.py:155` (`@flow_traced Flow.DATA_INFO`).
- Pipelined `redis_cache.mget(market_odds:{gid})`; cached entries with `_game_date` invalidate when game crosses `_is_live_window` (5 min before kickoff).
- Misses partition into live/pregame via `_lookup_game_info`.
- Live → `_fetch_live_kalshi_batch` `:401` reads `live_odds:{gid}`, `_is_payload_fresh`, `_extract_kalshi_from_payload`, `_apply_vig_to_summary`; misses *omitted* (not None) so Backend-Server `if bet_type in odds_data` doesn't TypeError.
- Pregame → `_compute_analytical(pipeline, gid, home, away)` `:552` (Poisson); cache write embeds `_game_date`.
- Live entries NOT written to `market_odds:*` (canonical key is `live_odds:{gid}`).
- `MarketOddsApiService` (`market_odds_api_service.py:83`) — DTOs used but no caller (cf. headline gap #4).

#### P-CLUSTER (NEW)
- Fit (P-FIT step): `PlayerClusterFitter.fit_all_players_with_priors` `cluster_models/fitter.py:254` over `zonal_repo` → `AllPlayerClusters` JSON cached `cluster_model:per_player_params` (TTL ~3600s).
- Blend (P-ZAP step): `_get_cluster_action_rate` `main_pipeline.py:1790` returns `(rate, variance, confidence, source ∈ {confirmed, predicted, missing, history})` via `PlayerClusterPredictor.predict_from_stats_with_lineup` `cluster_models/predictor.py:565`.
- In `_sample_player_counts` `:2376-2433`: alpha = `cluster_model_lineup_alpha` (0.7) for confirmed/predicted/missing else `cluster_model_blend_alpha` (0.3); `effective = (1−α)·zonal·rate + α·min(cluster_rate·zonal, 1)`.
- Variance cached in `_cluster_variance_cache[pid]` for vig widening.

#### P-REFIT — blue-green pipeline refit (NEW)
- F-HTTP-17 / F-HTTP-19 → `RefitManager.start_refit(group, force, calibrate, calibrate_player_props)` `refit_manager.py:67`.
- Per-group `threading.Lock` (lazy via `_locks_guard`); running → 409.
- Daemon `_refit_worker` → `_run_refit` `:144`: re-read seasons via `pipeline_config_repo.get_seasons_for_group` (Issue 1), build new `MainPipelineConfig` from `current_config.to_dict()` + overrides (Issue 2), `create_pipeline_fn(new_config).fit_models(force_refit=force)` (calls P-FIT).
- URA-T30: `_ensure_models_loaded() = False` → FAILED, no swap.
- `_validate_pipeline` `:273` runs `predict_match` smoke on 3 recent games (`home/away ∈ [0.1, 5.0]`, `n_teams ≥ 4`).
- Success → atomic `self.pipelines[group] = new_pipeline` → SWAPPED.

#### P-EXP — exposure-snapshot flush (NEW)
- F-TASK-K (`exposure_task` 300s, `app.py:325`) → `asyncio.to_thread(ExposureService.flush_snapshots)` `exposure_service.py:143`.
- `SCAN_ITER match=exposure:match:*` (cluster-safe per-node).
- Per match `HGETALL` → `(bet_count, total_stake, single_stake, parlay_stake)`; build `ExposureSnapshots` ORM row (var95/var99 placeholder 0.0); bulk INSERT.

#### P-LINEUP — lineup-change detection (NEW)
- F-TASK-J per game → `LineupChangeDetector.detect(new_lineup)` `lineup_change_detector.py:139`.
- GET Redis `lineup:{gid}` (24h TTL); set-diff over starters+bench (starter↔bench swaps count); on diff SET cache, return `(diff, True)` for caller to upsert + Kafka publish.

---

## Cross-class flow walks (end-to-end)

### W1: Parlay pricing with calibration (HTTP → service → pipeline → cache → Kafka)

Components: F-HTTP-22 (`POST /get_parlay_odds`) + P-PARLAY + P-SIM-CAL + P-CAL (offline writer) + F-TASK-L (analytics fan-out).

```
client
  └── POST /get_parlay_odds                               (odds.py:101, F-HTTP-22)
        ├── per-pick → GetOddsForBet.get_odds_for_<type>  (odds_service.py:743..1415, @flow_traced ODDS_COMPUTE)
        │     └── pipeline.get_odds_for_parlay_bet        (parlay_odds.py:1592, @traced)
        │           ├── parlay_validator.validate         (live_policy.py:383)   ── reject ended
        │           ├── live legs → _get_live_parlay_odds                       ── reads live_odds:{gid}
        │           └── pre-match legs:
        │                 ├── PricingContext(pipeline, gid)              (pricing_context.py)
        │                 │     └── _get_match_calibration → match_cal_adj:{gid}  (Redis, set by P-CAL/F-TASK-I)
        │                 ├── full_adj_fingerprint(adj) into cache key       (cache_fingerprint.py)
        │                 │     └── GET parlay_cache:{gid}:adj=<fp>           (TTL 30s)
        │                 └── miss → simulate_match_calibrated                (simulation.py:43, P-SIM-CAL)
        │                                 └── _apply_match_cal_lambda          (main_pipeline.py:270)
        ├── compute combined_odds, individual_odds, legs_details
        └── fire-and-forget odds_request_producer.emit(OddsRequestEvent ...)   (odds.py:181-194)
              └── Kafka odds.odds_requests   (acks=1, linger 50ms)
                    └── RequestAnalyticsService consumer  (request_analytics_service.py:54, F-TASK-L)
                          └── Redis HINCRBY req_analytics:match:{gid}, HLL pfadd, ZADD user_activity
```

The calibration adj that `PricingContext` reads is written by F-TASK-I + P-CAL on a separate 120s loop, so any one parlay request sees the most recent adj <120s ± cooldown old. The `full_adj_fingerprint` is what lets `parlay_cache` keep a 30s TTL without serving stale adjustments after a sweep.

### W2: League migration → cache invalidation → next odds request

Components: F-HTTP-21 (`POST /admin/migrate/{league_code}`) + `MigrationService` + post-migration cache invalidation + downstream P-PARLAY / P-ZAP cache misses.

```
operator
  └── POST /admin/migrate/{league_code}                  (pipeline_admin.py:659, F-HTTP-21)
        └── MigrationService.migrate_league              (info/migration_service.py:89)
              ├── _run_whoscored                          (:153)  ── invokes scripts/migrations/async_migrate_whoscored_new_to_postgres_v2.py
              │     └── INSERT/UPDATE lineups, predicted_lineup_players, missing_players, match_statistics
              │     └── INSERT row into migration_runs_audit
              ├── _run_sofascore                          (:212, gated on whoscored success unless force=True)
              │     └── similar script invocation
              └── post-completion cache invalidation       (per docstring; pattern delete on prob:zap:*, mom_odds:*, match_cal:*)
                    └── next /get_parlay_odds (W1) sees:
                          - PricingContext loads fresh adj  (cache invalidated)
                          - simulate_match_calibrated misses → re-runs (P-SIM-CAL key changes via team_lambda_fingerprint)
                          - parlay_cache misses → fresh price
```

This flow is the post-deploy verification path the data-health endpoint (F-HTTP-3 / F-HTTP-4) is designed to monitor — the alerts surfaced by `DataCompletenessService` are exactly the silent-zero failure modes a half-applied migration produces.

### W3: External odds ingest → live-priced odds

Components: F-TASK-H (W3 scheduled fetch) + Kafka `odds.market_updates` consumer + Redis `live_odds:{gid}` + P-LIVE read by F-HTTP-24..27.

```
F-TASK-H run_scheduled_odds_fetch (every 300s, gated)
  ├── _run_odds_api_fetch(market_categories)       (scheduled_odds_fetch.py:130)
  │     └── HTTP → Odds API → Postgres external_odds.market_odds
  ├── _run_kalshi_fetch                            (scheduled_odds_fetch.py:171)
  │     └── HTTP → Kalshi → Postgres + Redis live_odds:{gid}, market:odds:{gid}
  └── _run_player_prop_fetch                       (scheduled_odds_fetch.py:210)
        └── HTTP → Odds API → Postgres + Redis player_props:{gid}

           [ separately, upstream tbg-streaming pushes incremental updates ]
                                          ↓
                                    Kafka odds.market_updates
                                          ↓
                  KafkaOddsConsumer._process_batch              (consumer.py:228)
                    ├── _DESERIALIZE_ERRORS narrow except       (:87-92)   ── propagate KeyError/AttributeError
                    ├── _parse_game_id                          (:430)     ── kafka.match_id.parse_error counter
                    ├── unknown market_id                       (:502)     ── kafka.odds_update.unknown_market counter
                    ├── Redis SET live_odds:{gid} (TTL 30s/300s)
                    └── DB upsert MoneylineOdds/TotalsOdds/SpreadOdds

client → POST /get_zat_odds (F-HTTP-25)
  └── GetOddsForBet → _get_live_odds_from_redis                 (odds_service.py:449, P-LIVE)
        ├── GET live_odds:{gid}
        ├── if missing: NO_COVERAGE vs PENDING via _has_historical_odds (DB)
        ├── if age > LIVE_BETTING_POLICY.freshness_threshold_seconds: STALE
        └── LiveOddsService.get_team_props (live_odds_service.py:91) reformats with power-vig
```

The new OTEL counters in `KafkaOddsConsumer` (parse_error / deserialize_error / unknown_market) make the silent-drop class previously pre-PR observable on Grafana — operationally important the moment an upstream producer schema drifts.

### W4: Lineup scrape → calibration → priced ZAP

Components: F-TASK-J (W4) + `LineupChangeDetector` + `KafkaLineupProducer` + (downstream) P-CAL via F-TASK-I + P-ZAP.

```
F-TASK-J run_lineup_scraper (adaptive, base 120s, gated)
  ├── games_repo.get_upcoming_games(season_ids)         (tasks.py:915)
  └── per game (filtered by lookahead + last-scraped):
        ├── scrape_lineup_from_db                       (workers/lineup_scraper.py:33, reads DB lineups)
        ├── LineupChangeDetector.detect                 (application/service/lineup/lineup_change_detector.py:66)
        │     └── GET lineup:{gid} (24h TTL); set-diff over starters+bench
        ├── if changed:
        │     ├── _upsert_lineup_to_db                  (tasks.py:850)   ── DUPLICATES KafkaLineupConsumer._upsert_lineup
        │     └── KafkaLineupProducer.send              (lineup_producer.py:85)  ── topic odds.lineup_updates
        └── flush() once per cycle

   [ no live consumer of odds.lineup_updates — KafkaLineupConsumer.start() is never called ]

           Independently, F-TASK-I run_match_calibration_check picks up
           lineup-aware features through the pipeline state and emits
           match_cal_adj:{gid} which P-ZAP / P-PARLAY then read via
           PricingContext on the next request (see W1).
```

Practical effect: until a downstream consumer for `odds.lineup_updates` is wired, the Kafka publish step is best thought of as fan-out hook for future consumers — DB writes are what feed the rest of the system today.

---

## Component cross-references

| Component handle | Defined | Consumed by |
|---|---|---|
| F-TASK-I | `tasks.py:1133` | writes `match_cal_adj:{gid}` for P-CAL; read by P-ZAP, P-PARLAY, P-ZAP-BATCH, P-TEAMPROP |
| F-TASK-J | `tasks.py:813` | writes `lineups` table (consumed by P-FIT lineup model fit) + Kafka `odds.lineup_updates` (no live consumer) |
| F-TASK-H | `scheduled_odds_fetch.py:256` | writes `live_odds:{gid}`, `market:odds:{gid}`, `player_props:{gid}` consumed by P-LIVE, P-MARKET, P-CAL, P-PLAYER-CAL |
| F-TASK-L | `request_producer.py:55` + `request_analytics_service.py:54` | sources from F-HTTP-22 / F-HTTP-23 / F-HTTP-28; sinks to Redis aggregate keys |
| F-TASK-K | `app.py:349` + `exposure_service.py:143` | writes `ExposureSnapshots` table |
| P-CAL | `match_calibration_service.py:549` | sole writer of `match_cal_adj:{gid}` |
| P-SIM-CAL | `simulation.py:43` | sole writer of `odds:sim:calibrated:*` |
| `MarketOddsApiService` | `market_odds_api_service.py:83` | **no consumer in this PR** |
| `KafkaLineupConsumer` | `lineup_consumer.py:46` | defined but `.start()` not invoked |
