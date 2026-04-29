# PR #140 — HTTP / REST Flows

Branch: `dev-coverage-and-odds-impro`. file:line references are PR head.

## Summary
- New routes: 19
- Modified routes: 11 (instrumentation or import-only)
- Routes missing `flow_traced` on service entry: 17 (all admin/auth)
- Routes missing `ROUTE_TO_FLOW` entry: 17 (admin paths drop into the `""` flow)
- Newly registered routers in `app.py`: 0 (all mounted via `rest/routes.py`)

`auth.router` and `data_health.router` are new routers; both are registered in `infrastructure/api/rest/routes.py:34, 44`. `app.py` change is import-only — the only `app.include_router` calls are still `rest_router` (`app.py:430`) and `ws_router` (`app.py:431`).

Note on `ROUTE_TO_FLOW`: keys for admin/pipeline_admin routes lack the `/admin` prefix the router is actually mounted under (e.g. `("POST","/pipelines/{name}/refit")` vs the live URL `/admin/pipelines/...`). The two `data-health` keys are the only admin entries that include `/admin/` and therefore resolve. This is a pre-existing wiring bug exposed further by this PR but worth flagging — every admin/pipeline-admin handler logs with `flow=""`.

## Routes (grouped by router)

### `infrastructure/api/rest/routers/auth.py` — NEW FILE (added in PR)
| Method | Path | Status | file:line | Service call | Flow decorator | ROUTE_TO_FLOW |
|---|---|---|---|---|---|---|
| POST | /auth/admin/login | NEW | auth.py:35 | `get_admin_password()`, `create_access_token()` (jwt_helper) | none | missing |
| POST | /auth/admin/logout | NEW | auth.py:74 | none — clears cookie | none | missing |

Auth: public; sets `admin_token` HttpOnly cookie. No JWT guard (it's the issuer).

### `infrastructure/api/rest/routers/data_health.py` — NEW FILE
| Method | Path | Status | file:line | Service call | Flow decorator | ROUTE_TO_FLOW |
|---|---|---|---|---|---|---|
| GET | /admin/data-health/games/{game_id} | NEW | data_health.py:61 | `DataCompletenessService.check_game` (`application/service/info/data_completeness_service.py:70`) | none | present (`request_context.py:123`, `admin.operations`) |
| GET | /admin/data-health/incomplete | NEW | data_health.py:84 | `DataCompletenessService.list_incomplete_games` (`data_completeness_service.py`) | none | present (`request_context.py:124`) |

Auth: shared `Depends(verify_admin_token)` via mount (`rest/routes.py:44`).

### `infrastructure/api/rest/routers/admin.py` — MODIFIED (mostly net-new routes)
| Method | Path | Status | file:line | Service call | Flow decorator | ROUTE_TO_FLOW |
|---|---|---|---|---|---|---|
| POST | /admin/cache/invalidate-lineup/{game_id} | NEW | admin.py:113 | `RepositoryFactory.lineups.clear_cache_for_game` | none | missing |
| POST | /admin/cache/invalidate-predicted-lineup/{game_id} | NEW | admin.py:142 | `predicted_lineups.clear_cache_for_game` | none | missing |
| POST | /admin/cache/invalidate-missing-players/{game_id} | NEW | admin.py:161 | `missing_players.clear_cache_for_game` | none | missing |
| POST | /admin/cache/invalidate-game/{game_id} | NEW | admin.py:176 | repo + pipeline lookup caches + Redis pattern delete | none | missing |
| GET | /admin/cache/sizes | NEW | admin.py:273 | repo `cache_size()` reads | none | missing |
| POST | /admin/migration/run-targeted | NEW | admin.py:446 | `BackgroundTasks.add_task(_run_targeted_migration_in_process)` (admin.py:356, calls `scripts/migrations/async_migrate_whoscored_new_to_postgres_v2.py`) | none | missing |
| GET | /admin/migration/{job_id} | NEW | admin.py:490 | in-memory `_migration_jobs` dict (admin.py:338) | none | missing |
| GET | /admin/config | NEW | admin.py:536 | `list_runtime_flags()` | none | missing |
| POST | /admin/config/{name} | NEW | admin.py:546 | `set_runtime_flag()` (Redis) | none | missing |
| DELETE | /admin/config/{name} | NEW | admin.py:573 | `clear_runtime_flag()` | none | missing |
| GET | /admin/config/ui | NEW | admin.py:593 | Jinja2 template `admin_config.html` | none | missing |
| GET | /admin/qa/bypass_game_time | UNCHANGED | admin.py:47 | – | – | present (as `odds.override`) |
| POST | /admin/qa/bypass_game_time | UNCHANGED | admin.py:57 | – | – | present |
| PUT | /admin/config/{zone_group} | UNCHANGED | admin.py:70 | – | – | present (collides path-prefix with new `/admin/config/{name}` — disambiguated only by Pydantic body shape) |

Auth: router-level `Depends(verify_admin_token)` via mount (`rest/routes.py:40`); the `/config*` block additionally re-asserts the dep on each handler (admin.py:538,548,575,596).

### `infrastructure/api/rest/routers/pipeline_admin.py` — MODIFIED (3 new routes + 1 modified)
| Method | Path | Status | file:line | Service call | Flow decorator | ROUTE_TO_FLOW |
|---|---|---|---|---|---|---|
| POST | /admin/pipelines/{name}/assign-and-fit | NEW | pipeline_admin.py:187 | `repo.add_season` + `RefitManager.start_refit` + saga rollback | none | missing |
| POST | /admin/pipelines/{name}/refit | MODIFIED (added `calibrate_player_props` query) | pipeline_admin.py:377 | `RefitManager.start_refit(name, force, calibrate, calibrate_player_props)` (`application/service/calibration/refit_manager.py`) | none | partial — entry exists at `("POST","/pipelines/{name}/refit")` so it does NOT match `/admin/pipelines/...` |
| POST | /admin/pipelines/{name}/post-process | NEW | pipeline_admin.py:436 | `MatchStatisticsProcessor`, `PlayerStatisticsProcessor`, `ZonalStatisticsProcessor`, `TeamZonalStatisticsProcessor` (sync, bulk DB writes) | none | missing |
| POST | /admin/pipelines/{name}/calibrate-player-props | NEW | pipeline_admin.py:508 | `RefitManager.start_refit(force=False, calibrate_player_props=True)` | none | partial — same prefix-mismatch as `/refit` |
| GET | /admin/sources/actions | NEW | pipeline_admin.py:417 | returns `SOURCE_ACTIONS` constant | none | missing |
| POST | /admin/reload-season-map | UNCHANGED but adjacent | pipeline_admin.py:335 | `_reload_routing_map()` | none | missing |
| POST | /admin/migrate/{league_code} | NEW | pipeline_admin.py:659 | `MigrationService.migrate_league` (`application/service/info/migration_service.py:89`) → `_run_whoscored` (l.153) → `_run_sofascore` (l.212) → `migration_runs_audit` table | none | missing |
| GET/PUT/POST/DELETE | /admin/pipelines, /admin/pipelines/{name}, /admin/pipelines/{name}/config, /admin/pipelines/{name}/seasons, /admin/seasons{,/unassigned}, /admin/refit/status{,/{name}} | UNCHANGED | pipeline_admin.py:113-360 | – | – | partial (entries exist without `/admin/` prefix) |

Auth: router-level admin guard via mount (`rest/routes.py:42`).

### `infrastructure/api/rest/routers/odds.py` — MODIFIED (instrumentation only)
| Method | Path | Status | file:line | Service call | Flow decorator | ROUTE_TO_FLOW |
|---|---|---|---|---|---|---|
| POST | /get_parlay_odds | MODIFIED — emits `OddsRequestEvent` per leg | odds.py:101 (emit at 181-194) | `GetOddsForBet.get_odds_for_<type>` (`application/service/odds/odds_service.py:743/800/995/1050/1415`) + Kafka `odds_request_producer.emit` | service has `@flow_traced(Flow.ODDS_COMPUTE)` | present |
| POST | /get_parlay_expected_profit | MODIFIED — emits per game_id | odds.py:667 (emit at 837-855) | same service entries + producer | service-level `flow_traced` | present |
| POST | /get_player_odds | MODIFIED (import path only) | odds.py:240 | `odds_service.get_odds_for_zap` `@flow_traced(Flow.ODDS_COMPUTE)` (`odds_service.py:743`) | service | present |
| POST | /get_zat_odds | MODIFIED (import) | odds.py:319 | `get_odds_for_zat` `flow_traced` (`odds_service.py:800`) | service | present |
| POST | /get_team_prop_odds | MODIFIED (import) | odds.py:423 | `get_odds_for_team_prop` `flow_traced` (`odds_service.py:1050`) | service | present |
| POST | /get_zao_odds | MODIFIED (import) | odds.py:475 | `get_odds_for_zao` `flow_traced` (`odds_service.py:995`) | service | present |
| POST | /get_mom_odds | MODIFIED (import) | odds.py:531 | `get_odds_for_mom` `flow_traced` (`odds_service.py:1415`) | service | present |

Net-new HTTP behaviour in this file is fire-and-forget Kafka emit (`request_producer.py:55`); the producer is started in `app.py:294` and stopped at `app.py:387`.

### `infrastructure/api/rest/routers/builder.py` — MODIFIED
| Method | Path | Status | file:line | Service call | Flow decorator | ROUTE_TO_FLOW |
|---|---|---|---|---|---|---|
| POST | /builder/game/{game_id}/all-odds | MODIFIED — emits `OddsRequestEvent("all_odds")` after computing | builder.py:233 (emit `builder.py:294-307` per diff) | `BuilderService.get_all_odds_for_game` + `IdMappingRepository` + `odds_request_producer.emit` | none on builder service | present (`odds.compute`) |
| GET | /builder/config, /builder/actions, /builder/config/{zone_group} | UNCHANGED | builder.py:106,124,142 | – | – | present |

### `infrastructure/api/rest/routers/info.py` — MODIFIED (import path only)
| Method | Path | Status | file:line | Service call | Flow decorator | ROUTE_TO_FLOW |
|---|---|---|---|---|---|---|
| GET | /info/games/market-odds | MODIFIED (import only) | info.py:354 | `MarketOddsService.get_market_odds` `@flow_traced(Flow.DATA_INFO)` (`application/service/odds/market_odds_service.py:154`) | service | present |
| (other /info/* routes) | UNCHANGED — single-line import edit at info.py:9 | | | | | |

### `infrastructure/api/rest/routers/live.py` — MODIFIED (import path only, no behaviour change)
| Method | Path | Status | file:line | Service call | Flow decorator | ROUTE_TO_FLOW |
|---|---|---|---|---|---|---|
| GET | /live/is_live/{game_id} | UNCHANGED | live.py:58 | `LiveBettingService` (now `application/service/betting/live_betting_service.py`) | none | present |
| POST | /live/is_live_batch | UNCHANGED | live.py:75 | same | none | **missing** (pre-existing gap, not introduced by this PR) |
| GET | /live/available_markets/{game_id} | UNCHANGED | live.py:100 | same | none | present |
| PUT/DELETE | /live/override/{game_id}, GET /live/overrides, PUT/GET /live/simulate, GET /live/policy | UNCHANGED | live.py:144,157,170,191,232,242 | `live_policy.LIVE_BETTING_POLICY` + override store | none | present |

### `infrastructure/api/rest/routers/parameters.py` — MODIFIED (import path only)
All routes unchanged behaviourally. ROUTE_TO_FLOW entries already present.

## Flow walks (for non-trivial new flows)

### F-HTTP-1: Targeted migration (BS → BO)
- Entry: `infrastructure/api/rest/routers/admin.py:446` `POST /admin/migration/run-targeted`
- Steps:
  1. validate `game_ids` non-empty (`admin.py:465`)
  2. mint `job_id`, register `_migration_jobs[job_id]` (`admin.py:468-477`)
  3. `BackgroundTasks.add_task(_run_targeted_migration_in_process, ...)` (`admin.py:479`)
  4. background fn dynamically loads `scripts/migrations/async_migrate_whoscored_new_to_postgres_v2.py` (`admin.py:381-392`) and runs migration `main(Namespace(...))`
  5. on success calls cache invalidation helpers (per docstring `admin.py:457`)
- Side effects: writes Postgres (`lineups`, `predicted_lineup_players`, `missing_players`, `match_statistics` etc.), Redis pattern-delete on `mom_odds:*`, `prob:zap:*`, `match_cal:*`
- Status: NEW. No `flow_traced`; flow tag would be empty in logs.

### F-HTTP-2: League migration facade
- Entry: `pipeline_admin.py:659` `POST /admin/migrate/{league_code}`
- Steps:
  1. resolve `MigrationService` → `migrate_league(league_code, days_back, include_whoscored, include_sofascore, force)` (`application/service/info/migration_service.py:89`)
  2. `_run_whoscored` (`migration_service.py:153`) — invokes WhoScored migrator script, records run in `migration_runs_audit`
  3. `_run_sofascore` (`migration_service.py:212`) — gated on WhoScored success unless `force=True`
  4. 24h idempotency via audit table
- Side effects: large Postgres writes, Redis cache invalidations on completion
- Status: NEW.

### F-HTTP-3: Assign-and-fit saga
- Entry: `pipeline_admin.py:187` `POST /admin/pipelines/{name}/assign-and-fit`
- Steps:
  1. idempotency probe — return `noop` if season already in group + last refit `swapped` (`pipeline_admin.py:213-230`)
  2. snapshot prior assignment for rollback (`_find_any_assignment` `pipeline_admin.py:237`)
  3. `repo.add_season(...)` upsert (`pipeline_admin.py:240-248`)
  4. `RefitManager.start_refit(name, force, calibrate, calibrate_player_props)` (`pipeline_admin.py:259`); 409 → `_rollback_assignment` (`pipeline_admin.py:269`)
  5. `_wait_for_refit_terminal(...)` poll loop until terminal status or timeout → 504 + rollback
  6. on `failed` → rollback + structured error response; on success → `_reload_routing_map()`
- Side effects: Postgres `pipeline_group_seasons`, in-process model cache swap, in-memory `SEASON_TO_GROUP` reload
- Status: NEW.

### F-HTTP-4: Auth login (issuer)
- Entry: `auth.py:35` `POST /auth/admin/login`
- Steps: validate body password against `get_admin_password()` (`core/utils/jwt_helper.py`) → `create_access_token({"sub":"admin","role":"admin"})` → `JSONResponse` + `Set-Cookie admin_token` (HttpOnly, samesite=lax, secure=False)
- Notes: `secure=False` is intentional per docstring (TLS terminates upstream); duplicates Backend-Server's `/auth/admin/login` so the same JWT works against either service.
- Status: NEW.
