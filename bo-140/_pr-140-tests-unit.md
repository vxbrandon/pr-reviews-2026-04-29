---
title: "BO #140 — Sub-report A: unit tests added/modified"
pr: TBG-AI/Backend-Odds#140
scope: tests/unit/** (added or modified)
audited_at: 2026-04-29
---

# Sub-report A — Unit tests

Read-only mapping of every unit test file the PR adds or modifies to the
production module(s) it exercises. All file:line citations target the
working tree at `/Users/tbg/Desktop/programming/Dev-Agentic/Backend-Odds`
on the PR branch.

## Tally

- Added: 73 unit-test files (counting `__init__.py` package markers as 0).
- Modified: 0 (the 2 modified test files in this PR live outside `tests/unit/` and are covered in sub-report B).
- Empty package `__init__.py` files added under `tests/unit/`: 14 (structural only — no test bodies).
- Fixtures added: `tests/unit/rate_access_baseline.json`.

## A.1 Application services

| test file | tests added | code under test (`file:line`) | type |
|---|---|---|---|
| `tests/unit/application/test_market_odds_api_service.py:1` | parses `tests/fixtures/odds_api/upcoming_events.json`; mocks `LiveOddsAPI`/`HistoricalOddsAPI`; covers `fetch_upcoming_events`, `OddsApiAuthError`, `OddsApiRateLimitError` paths | `src/backend_odds/application/service/odds/market_odds_api_service.py:1` (450 added) | unit |
| `tests/unit/application/test_vig_policy.py:1` | golden snapshots over `vig_margin ∈ {0.05, 0.16}` × odds grid for `apply_divisive_vig`, `apply_power_vig`, `invert_power_vig` | `src/backend_odds/application/service/odds/vig_policy.py:1` (191 added) | unit |
| `tests/unit/application/test_data_completeness_service.py:1` | per-rule unit tests with hand-rolled `_FakeSession` that pattern-matches on SQL prefixes | `src/backend_odds/application/service/info/data_completeness_service.py:1` (338 added) | unit |
| `tests/unit/application/test_backtest_runner.py:1` | exercises `BacktestRunner` with `_FakeMainPipelineConfig`; patches `_build_pipeline_config` to avoid Redis/DB | `src/backend_odds/application/service/backtest/backtest_runner.py:1` (542 added) | unit |
| `tests/unit/test_team_props_calibration_service.py:1` | seeds corner/card markets, asserts `_calibrate_team_props` factor direction, clamp bounds, DB-error swallow, `predict_team_mean=None` skip | `src/backend_odds/application/service/calibration/match_calibration_service.py` (1989 added module) | unit |
| `tests/unit/test_team_props_calibration_parity.py:1` | parity between corner & card calibration paths (same factor on equivalent input) | same | unit |
| `tests/unit/test_match_calibration_unbounded_fit.py:1` | URA-T14: drives unbounded L-BFGS-B 2-D fit; asserts (A) gap-closing on 30/60% market vs base, (B) determinism within 1e-6, (C) extreme-factor structured WARN, (D) single `minimize` call (regression on coord-descent), (E) env-var bound `MATCH_CALIBRATION_FACTOR_MAX=2.0` truncates optimum | `match_calibration_service._optimise_factors` | unit |
| `tests/unit/test_match_calibration_fitted_r.py:1` | URA-T15: locks fitted moneyline residual into pricing | `match_calibration_service` | unit |
| `tests/unit/test_calibrate_player_props_from_redis.py:1` | round-trips `player_props:{gid}` payload shape into `_load_player_props_from_redis` | `match_calibration_service._load_player_props_from_redis` | unit |
| `tests/unit/test_extreme_factor_policy.py:1` | URA-T41: `clamp` (default), `skip`, `passthrough_with_alert` (with 1000-entry sorted-set cap), `block_pricing` (sentinel honoured by `get_prob_for_zap`) | `src/backend_odds/application/service/odds/extreme_factor_alerts.py:1` (140 added) + `match_calibration_service` | unit |
| `tests/unit/test_player_prop_factor_clamp.py:1` | T29 `[0.3, 3.0]` clamp boundaries (inclusive/exclusive) | `match_calibration_service` | unit |
| `tests/unit/test_player_prop_decoupled_calibration.py:1` | per-action calibration is independent (one action's factor doesn't bleed into another) | `match_calibration_service` | unit |
| `tests/unit/test_player_prop_end_to_end.py:1` | end-to-end input → adjustment → priced odds | `match_calibration_service` + main_pipeline | unit |
| `tests/unit/test_player_adjustments_propagate.py:1` | adjustments flow into `main_pipeline.get_prob_for_zap` | `match_calibration_service` + `main_pipeline.py` | unit |
| `tests/unit/test_player_prop_fetch_wrapper.py:1` | wrapper around scheduled-odds fetch (logging, error handling) | calibration helper | unit |
| `tests/unit/test_team_adjustments_consumer.py:1` | round-trip on team-adjust consumer | `match_calibration_service` + downstream | unit |
| `tests/unit/test_simulate_match_calibrated.py:1` | calibrated Monte-Carlo simulation outputs | pipeline + `match_calibration_service` | unit |
| `tests/unit/test_zao_calibrated.py:1` | ZAO odds when calibration is applied | `core/prediction_models/orchestration/zao_odds.py` + match_calibration | unit |
| `tests/unit/test_assign_and_fit_saga.py:1` | saga over assign + fit (covers `infrastructure/api/tasks.py` saga branch) | `src/backend_odds/infrastructure/api/tasks.py` (347 added) | unit |
| `tests/unit/test_admin_config_routes.py:1` | URA-T40 admin runtime-flag routes (GET/POST/DELETE + HTML UI); JWT auth dependency overridden, Redis monkey-patched at module load | `src/backend_odds/infrastructure/dependencies/admin_auth.py:1` (70 added) + `infrastructure/api/rest/routers/admin.py` | unit |
| `tests/unit/test_runtime_flags.py:1` | runtime-flag store CRUD with fake Redis | `src/backend_odds/core/utils/runtime_flags.py` (added, 481 lines on disk) | unit |
| `tests/unit/test_event_source_resolution.py:1` | source-priority resolution for `event_source` table | `src/backend_odds/core/data/event_source.py:1` (75 added) | unit |
| `tests/unit/test_source_selection_config.py:1` | config-driven event-source selection | `event_source.py` + repos | unit |
| `tests/unit/test_pricing_context.py:1` | `PricingContext` dataclass invariants | `core/prediction_models/orchestration/pricing_context.py` (added, 242 lines on disk) | unit |
| `tests/unit/test_market_odds_service_pricing_context.py:1` | `PricingContext` propagation through market-odds service | `application/service/odds/market_odds_service.py` (renamed/modified) | unit |
| `tests/unit/test_batch_zap_pricing_context.py:1` | batch ZAP context plumbing | `core/prediction_models/orchestration/batch_zap.py` | unit |
| `tests/unit/test_player_mapping.py:1` | player-name → internal id mapping helper | repo helpers | unit |
| `tests/unit/test_get_prob_for_zap_calibration.py:1` | `get_prob_for_zap` honours adjustment + pricing-block sentinel | `main_pipeline.get_prob_for_zap` | unit |
| `tests/unit/test_parlay_calibration_consistency.py:1` | parlay path uses same calibration as singles | `orchestration/parlay_odds.py` + `main_pipeline.py` | unit |
| `tests/unit/test_ensure_models_loaded.py:1` | URA-T29 ensure-models guard (early-fail when artifacts missing) | dependencies / model-load layer | unit |
| `tests/unit/test_rate_access_enforcement.py:1` | URA rate-access baseline pinned via `tests/unit/rate_access_baseline.json` | `core/data/data_access/*` rate access | unit |
| `tests/unit/test_joint_calibration_demo.py:1` | joint calibration demo path | `match_calibration_service` | unit |
| `tests/unit/test_migration_audit.py:1` | exercises `migration_runs_audit` table interactions | alembic-side audit table + `MigrationService`-adjacent code | unit |

## A.2 Repositories & data access

| test file | tests added | code under test (`file:line`) | type |
|---|---|---|---|
| `tests/unit/core/data/data_access/test_games_repository.py:1` | repo methods against `unit-test.invalid` URL (no real DB connection) | `src/backend_odds/core/data/data_access/games_repository.py:1` (+71/-2) | unit |
| `tests/unit/core/data/data_access/test_match_statistics_repository.py:1` | new fields, action coverage including `dribbles` | `match_statistics_repository.py:1` (+57/-12) | unit |
| `tests/unit/core/data/data_access/test_player_statistics_repository.py:1` | new fields, action coverage | `player_statistics_repository.py:1` (+89/-15) | unit |
| `tests/unit/core/data/data_access/test_players_repository.py:1` | players repository | `players_repository.py:1` (+55/-23) | unit |
| `tests/unit/core/data/data_access/test_zonal_statistics_repository.py:1` | zonal-stats reads | `zonal_statistics_repository.py:1` (+24/-4) | unit |

## A.3 Prediction models

| test file | tests added | code under test (`file:line`) | type |
|---|---|---|---|
| `tests/unit/core/prediction_models/zonal_models/test_dirichlet_per_action_alpha.py:1` | per-action α (URA-T18 fix) | `zonal_models/dirichlet.py` | unit |
| `tests/unit/core/prediction_models/zonal_models/test_force_refit_compat.py:1` | force-refit cache-compat path | `zonal_models/zonal_model_pipeline.py` | unit |
| `tests/unit/core/prediction_models/zonal_models/test_insufficient_teams_cache_guard.py:1` | guard against thin training data | `zonal_model_pipeline.py` | unit |
| `tests/unit/core/prediction_models/zonal_models/test_zonal_model_pipeline.py:1` | overall pipeline shape | `zonal_model_pipeline.py` | unit |
| `tests/unit/core/prediction_models/team_models/test_team_model_pipeline_tz.py:1` | timezone handling in team-model pipeline | `team_models/team_model_pipeline.py` | unit |
| `tests/unit/core/prediction_models/mom_models/test_mom_overround.py:1` | overround math | `mom_models/mom_model.py` | unit |
| `tests/unit/core/prediction_models/mom_models/test_mom_rng.py:1` | RNG determinism | `mom_models/mom_model.py` | unit |
| `tests/unit/core/prediction_models/test_config_utc.py:1` | config UTC normalisation | `prediction_models/config.py:1` (+111/-5) | unit |
| `tests/unit/core/prediction_models/player_action_models/test_player_action_pipeline_starter_filter.py:1` | locks the starter-only assumption (CLAUDE.md mandatory rule) | `player_action_models/player_action_pipeline.py:181` | unit |
| `tests/unit/core/prediction_models/player_action_models/test_no_dead_debug_logs.py:1` | grep-style: no `print()` / dead debug logs in prod code | source tree | unit (lint-style) |
| `tests/unit/core/prediction_models/player_action_models/test_no_hardcoded_debug_ids.py:1` | grep-style: no hardcoded player/game IDs | source tree | unit (lint-style) |

In-tree (lives under `src/`, also a unit suite):

| test file | tests added | code under test (`file:line`) | type |
|---|---|---|---|
| `src/backend_odds/core/prediction_models/tests/test_cluster_zone_distributions.py:1` (370 lines) | per-cluster zone distributions; cluster-zonal joint prediction (Change 1 + Change 2 in cluster models work) | `cluster_models/fitter.py:1` (698 added), `cluster_models/fitted_model.py:1` (282 added), `cluster_models/predictor.py:1` (717 added) | unit |
| `src/backend_odds/core/prediction_models/tests/test_zonal_model.py:1` (modified, 962 lines) | full zonal-model pipeline coverage | `zonal_models/*` | unit |

## A.4 Infrastructure (Kafka / cache / migrations / alembic)

| test file | tests added | code under test (`file:line`) | type |
|---|---|---|---|
| `tests/unit/infrastructure/cache/test_key_utils.py:1` | cluster-safe key helpers | `src/backend_odds/infrastructure/cache/key_utils.py:1` (49 added) | unit |
| `tests/unit/infrastructure/kafka/test_consumer_parse_game_id.py:1` | `parse_game_id` edge cases | `infrastructure/kafka/consumer.py` (+188/-6) | unit |
| `tests/unit/infrastructure/kafka/test_consumer_silent_failures.py:1` | consumer must not raise on poisoned messages | `consumer.py` | unit |
| `tests/unit/infrastructure/kafka/test_schemas.py:1` | schema round-trip | `infrastructure/kafka/schemas.py:1` (+122/-3) | unit |
| `tests/unit/alembic/test_env_safety_flags.py:1` | both `alembic/env.py` + `alembic_ext_odds/env.py` set `compare_type=True` and `compare_server_default=True` in offline + online paths | `alembic/env.py:1`, `alembic_ext_odds/env.py:1` | unit |
| `tests/unit/alembic/test_ext_odds_baseline_migration.py:1` | smoke for the 989-line baseline migration | `alembic_ext_odds/versions/f0a1b2c3d4e5_initial_ext_odds_schema.py` | unit |
| `tests/unit/migrations/test_drift_check.py:1` | hermetic alembic config in tmp dir + sqlite engine; locks `check_alembic_drift` (audit A.5) | `src/backend_odds/infrastructure/migrations/drift_check.py:1` (243 added) | unit |
| `tests/unit/migrations/test_sf_match_statistics_mapping.py:1` | SF → `sf_match_statistics` field mapping | `core/data/data_process/match_statistics.py` (+64/-8) + migration | unit |
| `tests/unit/migrations/test_sf_migrator_period.py:1` (374 lines) | period derivation in SF migrator | `core/data_scraping/period.py:1` (303 added) + `sqlalchemy/types.py:1` (61 added) | unit |
| `tests/unit/migrations/test_sf_migrator_own_goal.py:1` | own-goal handling | SF migrator | unit |
| `tests/unit/migrations/test_sf_migrator_no_incident_events.py:1` | empty-incidents path | SF migrator | unit |
| `tests/unit/migrations/test_sf_migrator_fallback.py:1` | fallback when SF missing | SF migrator | unit |
| `tests/unit/migrations/test_sf_migrator_batching_and_lock.py:1` | batch-lock semantics for `match_lock` table | SF migrator + match_lock alembic version | unit |

## A.5 Utilities & scripts

| test file | tests added | code under test (`file:line`) | type |
|---|---|---|---|
| `tests/unit/core/utils/test_atomic_io.py:1` | atomic write / fsync / rename | `src/backend_odds/core/utils/atomic_io.py:1` (added, 77 lines) | unit |
| `tests/unit/core/utils/test_atomic_io_model_integration.py:1` | model save sites use atomic_io | `atomic_io.py` + model save callers | unit |
| `tests/unit/core/utils/test_env.py:1` | env-helper coercion | `src/backend_odds/core/utils/env.py:1` (added, 31 lines) | unit |
| `tests/unit/data_scraping/test_period_sf_empirical.py:1` | empirical SF period strings (`HT/FT/ET/PEN` plus mixed-case) | `core/data_scraping/period.py:classify_period_marker`, `derive_period`, `build_period_segments` | unit |
| `tests/unit/scripts/test_entrypoint.py:1` | CLI entrypoint contract | repo entrypoint | unit |
| `tests/unit/scripts/test_verify_pricing_pipeline.py:1` | rematch detection, fetch freshness, divergence threshold | `scripts/odds_qa_testing/verify_pricing_pipeline.py` | unit |

## A.6 Conftest + fixture infrastructure (relevant for understanding hermeticism)

- `tests/unit/conftest.py:14-16` — sets `DATABASE_URL` to `postgresql://unit-test:unit-test@unit-test.invalid:5432/unit-test` so the data-access package `__init__` doesn't crash. Real DB connections fail loud.
- `tests/unit/__init__.py` — empty package marker (added).
- 13 additional empty `__init__.py` files added to mirror `tests/unit/` to the `src/backend_odds/` tree (`alembic/`, `application/`, `core/`, `core/data/`, `core/data/data_access/`, `core/prediction_models/`, `core/prediction_models/player_action_models/`, `core/prediction_models/zonal_models/`, `core/utils/`, `data_scraping/`, `infrastructure/`, `infrastructure/cache/`, `infrastructure/kafka/`, `migrations/`).

## A.7 Style / quality observations on the unit suite

1. `tests/unit/test_admin_config_routes.py:29-34` rewrites `RedisCacheRepository.__init__`/`get`/`set`/`delete` directly on the class at module-import time. This persists for the rest of the pytest session — any subsequent test in the same process that uses `RedisCacheRepository` will see the neutered class. Should be scoped via `monkeypatch` fixture or a `conftest.py` autouse with explicit teardown.
2. `tests/unit/core/prediction_models/player_action_models/test_no_dead_debug_logs.py:1` and `test_no_hardcoded_debug_ids.py:1` are grep-style (assert-on-file-content) tests. Useful as guard rails but they pass even if behaviour is broken.
3. `tests/unit/application/test_backtest_runner.py:62-70` patches `_build_pipeline_config` at module boundary to avoid `MainPipelineConfig.__init__()` touching Redis. The patch is correctly scoped to the test module, but tests under `tests/unit/test_assign_and_fit_saga.py` and `tests/unit/test_extreme_factor_policy.py` use `importlib.reload`-style patterns that risk leaking module state across tests; verify under `pytest-xdist`.
4. The whole match-calibration unit cluster (~15 files) mocks `predict_team_mean` and the SQLAlchemy session with `MagicMock`. CLAUDE rule (`.claude/rules/testing.md`) forbids DB mocks for *integration* tests; unit tests are fine, but the only end-to-end tests that wire this calibration into a real pipeline live in `tests/integration/` and require `--run-integration` (see sub-report B).
5. The `tests/unit/rate_access_baseline.json` fixture (12 lines) is a frozen baseline. Tests that assert against it must regenerate when rate-access intentionally changes; commit-message discipline matters.
