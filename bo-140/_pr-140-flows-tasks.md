# PR #140 — Scheduled + Kafka Flows

Branch: `dev-coverage-and-odds-impro` → `dev`. Read-only audit. Verified by reading head files in the PR branch.

## Summary
- New scheduled / background tasks: 6 (lineup scraper W4, match calibration W2, scheduled odds fetch W3, market-odds warm-up, exposure snapshot flush, scheduled calibration is preexisting but rewired)
- New kafka producers: 2 (`odds.lineup_updates`, `odds.odds_requests`); 1 producer module (`KafkaOddsProducer` for `odds.market_updates`) is unchanged at the producer site.
- New kafka consumers: 2 process-internal (`RequestAnalyticsService` on `odds.odds_requests`; `KafkaLineupConsumer` retained but **NOT started** in lifespan). `KafkaOddsConsumer` modified.
- Modified: `KafkaOddsConsumer` (BATCH_SIZE 2000→200, narrow-except, dedup metrics for parse/deserialize/unknown-market).
- Ralph loop: NO scripts in this PR — only docs (`docs/pr/ralph-loop-agents-guide.md`, `AI_DOCS/reference/2026-04-18_ralph-audit-fix-report.md`). v1 report flagging "ralph wiring" is wrong.
- v1 cross-checks: lineup-scraper W4, match-calibration W2, scheduled-odds-fetch W3, exposure flush, `odds.odds_requests` producer, analytics consumer pair — all confirmed against code.

## Scheduled tasks (lifespan-registered)
| Name | Trigger / interval | file:line | Service call | Status (env-gate) | Idempotent? |
|---|---|---|---|---|---|
| `run_odds_broadcast` | loop, 60s | `web/app.py:218`; impl `api/tasks.py:41` | `global_broadcaster.broadcast_to_clients(process_odds_for_client, ...)` `tasks.py:76` | Always on | Yes — read-only WS push per cycle |
| `run_live_odds_broadcast` | loop, `LIVE_BETTING_POLICY.broadcast_interval_seconds` (10s default) | `app.py:221`; impl `tasks.py:97` | same broadcaster, live-only filter | Always on | Yes |
| `run_spotlight_broadcast` | loop, 120s | `app.py:230`; impl `tasks.py:232` | `connection_manager.generate_spotlight_odds_data(season_ids)` then parallel send | Always on | Yes |
| `run_idle_cleanup` | loop, 300s | `app.py:237`; impl `tasks.py:400` | disconnects clients idle >30 min | Always on | Yes (idempotent disconnect) |
| `run_expected_profit_task` | loop, 5 min | `app.py:240`; impl `tasks.py:451` | reads `active_parlays:monitoring`, writes `parlay_profit_metrics` Redis key (TTL 7200s) | Always on | Yes — Redis SET ex |
| `run_scheduled_calibration` | loop, `CALIBRATION_INTERVAL_HOURS` (24h default; sleeps one interval first) | `app.py:243-244`; impl `tasks.py:1041` | `refit_manager.start_refit(group, force, calibrate)` per pipeline; polls 30s up to 30 min | Gated by `CALIBRATION_ENABLED=true` (`tasks.py:797`) | Yes — `start_refit` returns conflict and is skipped |
| `_warm_market_odds_loop` | loop, 240s | `app.py:256-278` | `MarketOddsService.warm_cache()` → Redis | Always on | Yes — overwrites cache |
| `run_scheduled_odds_fetch` (W3) | loop, `ODDS_FETCH_INTERVAL_SEC` (300s) | `app.py:292`; impl `kafka/scheduled_odds_fetch.py:256` | calls `fetch_all_market_odds.fetch_and_store_all_odds`, `fetch_kalshi_odds.fetch_and_store(write_redis=True)`, `fetch_player_prop_odds.fetch_and_store_all_player_props` via `asyncio.to_thread` | Gated by `ODDS_FETCH_ENABLED=true` (`scheduled_odds_fetch.py:71`) | **Partial** — DB writes use upserts, but no per-cycle dedup guard; concurrent cycles guarded only by sequential `await` chain |
| `run_match_calibration_check` (W2) | loop, `MATCH_CALIBRATION_CHECK_INTERVAL_SEC` (120s) | `app.py:305`; impl `tasks.py:1133` | `MatchCalibrationService.calibrate_match(...)` per upcoming game via `asyncio.to_thread` | Gated by `MATCH_CALIBRATION_ENABLED` | Yes — service has cooldown (`MATCH_CALIBRATION_COOLDOWN_SEC`) and divergence check |
| `run_lineup_scraper` (W4) | loop, `LINEUP_SCRAPER_INTERVAL_SEC` (120s) + adaptive (skip >24h, 30 min @ 2-24h, base @ <2h/live) | `app.py:329`; impl `tasks.py:813` | `scrape_lineup_from_db` → `_upsert_lineup_to_db` (`tasks.py:850`) → `KafkaLineupProducer.send` → flush per cycle | Gated by `LINEUP_SCRAPER_ENABLED` (`tasks.py:808`) | Yes — `LineupChangeDetector` short-circuits unchanged lineups; DB upsert by (game_id, player_id) |
| `_exposure_snapshot_loop` | loop, sleep-first 300s | `app.py:349-366` | `ExposureService.flush_snapshots()` via `asyncio.to_thread` (`exposure_service.py:143`) — SCAN `exposure:match:*`, write to `ExposureSnapshots` | Always on | Idempotent per-(match, snapshot_at) row-level upsert (verify when reviewing exposure_service body) |
| Pre-existing `run_parameter_backup_task` | loop, 20 min | `tasks.py:341` (defined) | NOT started in current `app.py` lifespan |  | n/a |
| Redis pub/sub `redis_subscriber_task` | listener | `tasks.py:145` | commented out at `app.py:234` | DISABLED | n/a |

Lifespan teardown cancels: odds, live_odds, spotlight, profit, calibration, market_odds, odds_fetch, exposure, match_cal, lineup_scraper (`app.py:372-374`). `idle_cleanup_task.cancel()` is **after** `await ... gather` and is not awaited — minor leak on shutdown.

## Kafka producers (table)
| Topic | Producer call site | file:line | Trigger | Schema |
|---|---|---|---|---|
| `odds.lineup_updates` (`KAFKA_LINEUP_TOPIC`) | `KafkaLineupProducer.send(lineup_update)` then `producer.flush()` | producer `kafka/lineup_producer.py:85,101`; call sites `tasks.py:1008,1021` | Each lineup-scraper cycle that detects a change | `LineupUpdate` JSON; key `lineup:{game_id}` |
| `odds.odds_requests` (`ODDS_REQUEST_KAFKA_TOPIC`) | `odds_request_producer.emit(OddsRequestEvent(...))` | producer `kafka/request_producer.py:128`; call sites `rest/routers/odds.py:186,848`, `rest/routers/builder.py:297` | Inline on REST odds-fetch endpoints; fire-and-forget (non-blocking, drops on broker outage) | `OddsRequestEvent` (user_id, game_id, market_types, bet_types, ts_ms, session_id, device_id) `request_producer.py:27` |
| `odds.market_updates` (preexisting `KAFKA_ODDS_TOPIC`) | unchanged in this PR (consumer side modified) | n/a in this PR | upstream (tbg-streaming) | `OddsUpdate` |

Producer config: lineup uses `enable.idempotence=true, acks=all, lz4` (`lineup_producer.py:23`); request producer uses `acks=1, linger.ms=50` (`request_producer.py:87`) — analytics-grade durability.

## Kafka consumers (table)
| Topic | Consumer file:line | Handler | Side effects | Manual span? |
|---|---|---|---|---|
| `odds.market_updates` | `kafka/consumer.py:228` (`KafkaOddsConsumer`); started `app.py:316` unless `--disable-kafka` | `_process_batch` parses `OddsUpdate`, splits moneyline/totals/spreads, writes Redis `live_odds:{game_id}` then upserts `MoneylineOdds/TotalsOdds/SpreadOdds` | Redis write (TTL 30s live / 300s prematch) + DB upsert; commits offsets sync | Yes — `kafka.process_batch`, `kafka.redis_write`, `kafka.db_upsert` (`consumer.py:379,513,519`); 3 OTEL counters (`kafka.match_id.parse_error`, `kafka.odds_update.deserialize_error`, `kafka.odds_update.unknown_market`) `consumer.py:70-97` |
| `odds.odds_requests` | `application/service/info/request_analytics_service.py:54`; started `app.py:344` unless `--disable-kafka` | `_aggregate_event` → Redis HINCRBY `req_analytics:match:{gid}` (TTL 86400), HLL `pfadd` for unique users, ZADD `req_analytics:user_activity` (TTL 7d) | Redis only; commits offsets sync | Yes — `request_analytics.process_batch` (`request_analytics_service.py:154`) |
| `odds.lineup_updates` | `kafka/lineup_consumer.py:46` (`KafkaLineupConsumer`) — **module retained, NOT started** in `app.py` lifespan; docstring at `lineup_consumer.py:1-12` confirms scraper now writes DB directly | upserts `LineupModel` rows | n/a (not running) | Defined `kafka.lineup.process_batch` `lineup_consumer.py:154` |

## Flow walks for new tasks

### F-TASK-1: Lineup Scraper (W4)
- Trigger: lifespan `asyncio.create_task(run_lineup_scraper(...))` `app.py:329`. Adaptive base interval 120s.
- Steps:
  - `games_repo.get_upcoming_games(season_ids, limit=100)` `tasks.py:915`
  - filter by lookahead window + per-game elapsed-since-last-scrape `tasks.py:923-957`
  - `scrape_lineup_from_db(session, game_id)` via `asyncio.to_thread` `tasks.py:975-979`
  - `LineupChangeDetector.detect(...)` (skip if unchanged) `tasks.py:988`
  - `_upsert_lineup_to_db` direct ORM upsert `tasks.py:850-888,997`
  - `KafkaLineupProducer.send` then `flush()` per cycle `tasks.py:1008,1021`
- Side effects: writes `lineups` table; publishes Kafka.
- Failure mode: per-game try/except logs and continues; `producer.close()` on cancel; **no retry/DLQ** for failed Kafka deliveries beyond producer's own idempotent retries.

### F-TASK-2: Match Calibration Check (W2)
- Trigger: lifespan `app.py:305`. Loop every 120s.
- Steps: `games_repo.get_upcoming_games(...)` → per game `cal_service.calibrate_match(game_id, home, away, trigger_type="market_odds")` via `asyncio.to_thread` `tasks.py:1188`.
- Side effects: writes Redis adjustment keys (managed by `MatchCalibrationService`).
- Failure mode: outer `try/except` logs sweep error and continues; cooldown enforced inside the service.

### F-TASK-3: Scheduled External Odds Fetch (W3)
- Trigger: lifespan `app.py:292`; loop `ODDS_FETCH_INTERVAL_SEC` (300s) `scheduled_odds_fetch.py:309`.
- Steps: sequential `_run_odds_api_fetch(market_categories)` → `_run_kalshi_fetch()` → `_run_player_prop_fetch()` each via `asyncio.to_thread`.
- Side effects: DB writes (odds tables); Redis `live_odds:{game_id}` (Kalshi) and `player_props:{game_id}` (player props). Also mutates `sys.path` once `_ensure_script_imports` `scheduled_odds_fetch.py:97`.
- Failure mode: per-stage `try/except logger.exception`; cycle continues. No alerting on quota exhaustion.
- **Idempotency risk:** the cap `ODDS_FETCH_PLAYER_PROPS_MAX_EVENTS=10` per league prevents quota blow-out, but consecutive cycles will always re-hit Odds API — DB writes assumed upsert-safe by called scripts. No mutex if previous cycle still running (sequential `await` chain in single task is enough today; would break if moved to threadpool fanout).

### F-TASK-4: Exposure Snapshot Flush
- Trigger: `_exposure_snapshot_loop` `app.py:349`. Loop sleeps 300s **before** first run.
- Steps: `exposure_service.flush_snapshots()` via `asyncio.to_thread`; scans `exposure:match:*` (cluster-safe SCAN), writes `ExposureSnapshots` rows.
- Failure mode: `try/except logger.warning`; per-cycle continues.

### F-TASK-5: `odds.odds_requests` producer/consumer pair (W1)
- Trigger:
  - producer started in lifespan `app.py:339`; `emit()` called inline in REST handlers (3 sites).
  - analytics consumer started unless `--disable-kafka` `app.py:343-344`.
- Side effects: Kafka publish (acks=1, linger 50ms); Redis aggregations on consume.
- Failure mode: producer drops on `BufferError`/`Exception` with WARN log `request_producer.py:146-149`; consumer offsets committed per batch (sync), errors logged + 1 s sleep `request_analytics_service.py:159-161`. Gated by `ODDS_REQUEST_TRACKING_ENABLED=true` (`request_producer.py:69`).

## Findings worth flagging
1. `idle_cleanup_task` is cancelled (`app.py:397`) but never awaited — pending coroutine on shutdown.
2. `KafkaLineupConsumer` is dead code in this PR's wiring (only producer used) — deletion candidate or wire it for downstream.
3. `run_scheduled_calibration` polls `refit_manager.get_status` every 30 s up to 1800 s with no exponential backoff — fine, just notable for review.
4. `run_scheduled_odds_fetch` ignores all repo arguments (`scheduled_odds_fetch.py:271-276`) — interface noise that should be cleaned up or removed.
5. `_warm_market_odds_loop` has no env gate — unconditional Redis writes every 240s, even in dev.
6. v1 report's "ralph loop wiring" claim is unsupported; the only ralph artifacts are markdown docs.
