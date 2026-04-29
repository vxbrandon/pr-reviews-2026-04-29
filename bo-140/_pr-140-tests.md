---
title: "BO #140 — Test Coverage Audit (synthesis)"
pr: TBG-AI/Backend-Odds#140
branch: dev-coverage-and-odds-impro → dev
state: OPEN
audited_at: 2026-04-29
auditor: tests-coverage-specialist
sub_reports:
  - outputs/_pr-140-tests-unit.md          # sub-report A
  - outputs/_pr-140-tests-integration.md   # sub-report B
  - outputs/_pr-140-tests-gaps.md          # sub-report C
---

# PR #140 — Test Coverage Audit

Read-only audit. Citations target the working tree at
`/Users/tbg/Desktop/programming/Dev-Agentic/Backend-Odds` on the PR branch.
This document synthesises three sub-reports listed above.

> **Note on sub-agent spawning.** The team-lead requested three parallel
> Opus sub-agents via the Agent tool. That tool is not present in this
> environment (no Agent / Task spawn primitive is exposed — only
> SendMessage to teammates and bash). I instead produced the three
> sub-reports as separate artifacts (`_pr-140-tests-{unit,integration,gaps}.md`)
> and synthesised them into this file, preserving the requested structure.

## 1. Headline numbers

- Test files **added**: 79 (73 under `tests/unit/`, 4 under `tests/integration/` plus 1 conftest, 3 under `tests/flow_tests/`, 1 under `tests/pr102/`, 1 at `tests/` root, 2 in-tree under `src/backend_odds/core/prediction_models/tests/`). See sub-report A and B.
- Test files **modified**: 2 (`tests/test_market_odds_service_live_kalshi.py:16` and `tests/qa/player_calibration/test_player_calibration.py:481`, both 1-line touches).
- Test fixtures added: 2 JSON (`tests/fixtures/odds_api/upcoming_events.json`, `tests/unit/rate_access_baseline.json`).
- Production source files **added or modified** (excl. `alembic/versions`, `scripts/_archive`, docs): **96 unique `.py` files** under `src/`. 47 added or modified more than 50 lines.
- Production source files with **no test importer** (per the import-grep in sub-report C): **42** (~3-5 false positives due to dynamic imports).
- Production source files with **>50 added LOC and no test importer**: **14**.
- New CI signals introduced by this PR: **0**. No new GitHub Actions workflow, no new pytest invocation, no new marker, no coverage threshold. See §5.

## 2. Implemented — test ↔ code map

The full per-test entries (assertions described, types, gating) live in
**sub-report A** (unit) and **sub-report B** (integration / flow / e2e).
Digest below.

### 2.1 Application services (sub-report A.1)

| test file | code under test (`file:line`) | type |
|---|---|---|
| `tests/unit/application/test_market_odds_api_service.py:1` | `application/service/odds/market_odds_api_service.py:1` (450 added) | unit |
| `tests/unit/application/test_vig_policy.py:1` | `application/service/odds/vig_policy.py:1` (191 added) | unit |
| `tests/unit/application/test_data_completeness_service.py:1` | `application/service/info/data_completeness_service.py:1` (338 added) | unit |
| `tests/unit/application/test_backtest_runner.py:1` | `application/service/backtest/backtest_runner.py:1` (542 added) | unit |
| 12 calibration files: `test_team_props_*`, `test_match_calibration_*`, `test_player_prop_*`, `test_extreme_factor_policy.py`, `test_calibrate_player_props_from_redis.py`, `test_simulate_match_calibrated.py`, `test_zao_calibrated.py`, `test_player_adjustments_propagate.py`, `test_team_adjustments_consumer.py`, `test_get_prob_for_zap_calibration.py`, `test_parlay_calibration_consistency.py`, `test_joint_calibration_demo.py` | `application/service/calibration/match_calibration_service.py` (1989 added) + `core/prediction_models/orchestration/main_pipeline.py:get_prob_for_zap` + parlay/zao/batch_zap | unit |
| `tests/unit/test_assign_and_fit_saga.py:1` | `infrastructure/api/tasks.py:1` (+347 / −3) | unit |
| `tests/unit/test_admin_config_routes.py:1` + `tests/unit/test_runtime_flags.py:1` | `infrastructure/dependencies/admin_auth.py:1` (70 added) + `core/utils/runtime_flags.py:1` (481 disk LOC) + admin router (dynamic import) | unit |
| `tests/unit/test_event_source_resolution.py:1` + `test_source_selection_config.py:1` | `core/data/event_source.py:1` (75 added) | unit |
| `tests/unit/test_pricing_context.py:1`, `test_market_odds_service_pricing_context.py`, `test_batch_zap_pricing_context.py` | `core/prediction_models/orchestration/pricing_context.py` (242 disk LOC) + market-odds + batch-zap | unit |
| `tests/unit/test_ensure_models_loaded.py:1` | model-load layer | unit |
| `tests/unit/test_rate_access_enforcement.py:1` | data-access rate access (with frozen baseline fixture) | unit |
| `tests/unit/test_migration_audit.py:1` | `migration_runs_audit` table | unit |

### 2.2 Repositories & data access (sub-report A.2)

| test file | code under test (`file:line`) | type |
|---|---|---|
| `tests/unit/core/data/data_access/test_games_repository.py:1` | `games_repository.py:1` (+71 / −2) | unit |
| `tests/unit/core/data/data_access/test_match_statistics_repository.py:1` | `match_statistics_repository.py:1` (+57 / −12) | unit |
| `tests/unit/core/data/data_access/test_player_statistics_repository.py:1` | `player_statistics_repository.py:1` (+89 / −15) | unit |
| `tests/unit/core/data/data_access/test_players_repository.py:1` | `players_repository.py:1` (+55 / −23) | unit |
| `tests/unit/core/data/data_access/test_zonal_statistics_repository.py:1` | `zonal_statistics_repository.py:1` (+24 / −4) | unit |

### 2.3 Prediction models (sub-report A.3)

| test file | code under test (`file:line`) | type |
|---|---|---|
| `tests/unit/core/prediction_models/zonal_models/test_dirichlet_per_action_alpha.py:1` | `zonal_models/dirichlet.py` | unit |
| `tests/unit/core/prediction_models/zonal_models/test_force_refit_compat.py:1` | `zonal_models/zonal_model_pipeline.py` | unit |
| `tests/unit/core/prediction_models/zonal_models/test_insufficient_teams_cache_guard.py:1` | `zonal_model_pipeline.py` | unit |
| `tests/unit/core/prediction_models/zonal_models/test_zonal_model_pipeline.py:1` | `zonal_model_pipeline.py` | unit |
| `tests/unit/core/prediction_models/team_models/test_team_model_pipeline_tz.py:1` | `team_models/team_model_pipeline.py` | unit |
| `tests/unit/core/prediction_models/mom_models/test_mom_overround.py:1` + `test_mom_rng.py:1` | `mom_models/mom_model.py` | unit |
| `tests/unit/core/prediction_models/test_config_utc.py:1` | `prediction_models/config.py:1` (+111 / −5) | unit |
| `tests/unit/core/prediction_models/player_action_models/test_player_action_pipeline_starter_filter.py:1` | `player_action_models/player_action_pipeline.py:181` | unit |
| `tests/unit/core/prediction_models/player_action_models/test_no_dead_debug_logs.py:1` + `test_no_hardcoded_debug_ids.py:1` | source tree (lint-style) | unit |
| `src/backend_odds/core/prediction_models/tests/test_cluster_zone_distributions.py:1` (370 lines) | `cluster_models/{fitter,fitted_model,predictor}.py` (1697 added) | unit (in-tree) |
| `src/backend_odds/core/prediction_models/tests/test_zonal_model.py:1` (962 lines, modified) | `zonal_models/*` | unit (in-tree) |

### 2.4 Infrastructure (sub-report A.4)

| test file | code under test (`file:line`) | type |
|---|---|---|
| `tests/unit/infrastructure/cache/test_key_utils.py:1` | `infrastructure/cache/key_utils.py:1` (49 added) | unit |
| `tests/unit/infrastructure/kafka/test_consumer_parse_game_id.py:1`, `test_consumer_silent_failures.py`, `test_schemas.py` | `infrastructure/kafka/consumer.py` (+188 / −6) + `schemas.py:1` (+122 / −3) | unit |
| `tests/unit/alembic/test_env_safety_flags.py:1` | `alembic/env.py:1`, `alembic_ext_odds/env.py:1` | unit |
| `tests/unit/alembic/test_ext_odds_baseline_migration.py:1` | `alembic_ext_odds/versions/f0a1b2c3d4e5_initial_ext_odds_schema.py` (989 lines) | unit |
| `tests/unit/migrations/test_drift_check.py:1` | `infrastructure/migrations/drift_check.py:1` (243 added) | unit |
| `tests/unit/migrations/test_sf_match_statistics_mapping.py:1` | SF → `sf_match_statistics` mapping + migration | unit |
| `tests/unit/migrations/test_sf_migrator_*.py` (5 files) | SF migrator + `core/data_scraping/period.py:1` (303 added) + `sqlalchemy/types.py:1` (61 added) | unit |

### 2.5 Utilities & scripts (sub-report A.5)

| test file | code under test (`file:line`) | type |
|---|---|---|
| `tests/unit/core/utils/test_atomic_io.py:1` + `test_atomic_io_model_integration.py:1` | `core/utils/atomic_io.py:1` (77 disk LOC) | unit |
| `tests/unit/core/utils/test_env.py:1` | `core/utils/env.py:1` (31 disk LOC) | unit |
| `tests/unit/data_scraping/test_period_sf_empirical.py:1` | `core/data_scraping/period.py:classify_period_marker / derive_period / build_period_segments` | unit |
| `tests/unit/scripts/test_entrypoint.py:1` | repo CLI entrypoint | unit |
| `tests/unit/scripts/test_verify_pricing_pipeline.py:1` | `scripts/odds_qa_testing/verify_pricing_pipeline.py` | unit |

### 2.6 Integration / flow / e2e (sub-report B)

| test file | code under test (`file:line`) | type | gating |
|---|---|---|---|
| `tests/integration/test_calibration_regression.py:1` (692 lines) | `match_calibration_service`, `main_pipeline.get_prob_for_zap`, ZAP cache layers | integration (real Postgres + Redis + subprocess'd FastAPI on port 11002) | `--run-integration` + game `371402987` fixture |
| `tests/integration/test_data_availability.py:1` (573 lines) | `match_calibration_service._load_*`, `infrastructure/kafka/scheduled_odds_fetch.py:1` (350 added) | integration (read-only) | `--run-integration` + ≥2,000 player-mapping rows |
| `tests/integration/test_data_completeness.py:1` (272 lines) | `application/service/info/data_completeness_service.py:1` (338 added) | integration (real Postgres seed + cleanup) | `--run-integration` |
| `tests/integration/test_model_quality.py:1` (778 lines) | base models + `match_calibration_service.calibrate_match` | integration (in-process, no subprocess) | `--run-integration` + cached models |
| `tests/flow_tests/test_sf_coverage_flows.py:1` (286 lines) | `pipeline_admin.py:1` (+448 / −1), `data_process/match_statistics.py` (+64 / −8) | flow | `flow-tests.yml --ci` |
| `tests/flow_tests/test_migrate_league_flow.py:1` (300 lines) | `application/service/info/migration_service.py:1` (268 added, monkey-patched) | flow | `flow-tests.yml --ci` |
| `tests/flow_tests/test_season_map_invalidation.py:1` (113 lines) | season-map plumbing | flow | `flow-tests.yml --ci` |
| `tests/test_odds_pricing_flows.py:1` (1594 lines) | `infrastructure/kafka/{request_producer,scheduled_odds_fetch}.py`, `lineup_models/*` | unit-style despite location | uncovered by any workflow |
| `tests/pr102/test_e2e_kafka_betting_flow.py:1` (399 lines) | cross-service auth + bet + Kafka topics + Redis analytics | **CLI script — pytest collects 0 from it** | manual only |

## 3. Missing — high impact

(Full annotated list with status / LOC / disk size in **sub-report C**.)

- **Lineup-models / lineup-Kafka pipeline** — six modules (`lineup_consumer.py` 267, `lineup_producer.py` 123, `lineup_scraper.py` 97, `lineup_change_detector.py` 159, `lineup_training_repository.py` 297, `lineup_source_priority.py` 99) plus `lineup_models/{constants,fitted_model,schemas,params}.py` (~830 disk LOC). **~1,870 LOC with no test importer.** Lineup events drive the starter-only assumption; silent corruption corrupts every priced ZAP.
- **Scheduled-fetch / analytics pipeline** — `scheduled_odds_fetch.py` (350) is the only producer of `live_odds:{gid}` and `player_props:{gid}`; integration test only checks outputs, not the loop's retry / backoff / poison-message paths. `migration_service.py` (268) has a flow test that monkey-patches its internals. `request_analytics_service.py` (255) and `exposure_service.py` (210) each have one distant importer in `tests/test_odds_pricing_flows.py` but no focused unit. `infrastructure/web/app.py:1` (+112 / −5) startup wiring is not asserted.
- **Trust-root JWT helpers** — `core/utils/jwt_helper.py` (95 disk LOC) is **mocked out** in the only test that touches admin auth (`tests/unit/test_admin_config_routes.py:32-34`). The trust root is unverified.
- **Largest single rewrite without a test** — `core/data/data_access/mom_repository.py` (+88 / −63, 601 disk LOC). MoM markets feed user-facing odds.
- **`pipeline_admin.py`** (+448 / −1) — exercised via flow test for a few endpoints; bulk of new endpoints lack negative-path unit coverage.
- **`infrastructure/api/tasks.py`** (+347 / −3) — saga path is covered by `test_assign_and_fit_saga.py`, but new lineup-producer start/stop, scheduled-odds tick, and calibration-scheduling glue are not unit-pinned.

## 4. Missing — lower impact

(See sub-report C.3 for the full list.)

- `core/utils/datetime_utils.py` (62 disk LOC) — no test.
- `infrastructure/dependencies/ws_dependencies_copula.py` (+22 / −4) — no test.
- `infrastructure/api/rest/routers/odds.py` (+40 / −2), `routes.py` (+28 / −4), `routers/data_health.py` (added, 111 disk), `routers/auth.py` (added, 79 disk) — routers untested as importers.
- `infrastructure/api/websocket/{manager,manager_typed,handler}.py` — 1-line changes each; no test imports them. CLAUDE.md flags these as critical.
- `infrastructure/web/middleware/request_context.py` (+3 / 0) — `ROUTE_TO_FLOW` map. No test asserts new routes registered.
- `infrastructure/cache/cache_utils.py` (+11 / 0) — no importer.
- Lint-style guard tests (`test_no_dead_debug_logs.py`, `test_no_hardcoded_debug_ids.py`) pass even when behaviour is broken.

## 5. Quality concerns

(See sub-reports A.7 and B.7 for details.)

1. `tests/pr102/test_e2e_kafka_betting_flow.py:1` is named like a pytest file but has no `def test_*` — pytest collects 0 from it. Either rename to `scripts/manual_qa/` or convert to parametrised pytest with `@pytest.mark.e2e`.
2. The two modified test files are 1-line touches (rename / import-path), not new coverage.
3. `tests/integration/test_calibration_regression.py:237-273` spawns a child Python process. Cleanup is `os.killpg(os.getpgid(...))` but no `try/finally` around the launch — non-pytest exceptions can leak the child. The 32-s `SLEEP_AFTER_ADJ_CHANGE_SEC` makes the suite ~2 min; flaky-by-time on slow runners.
4. Skip-paths in two integration tests use `if not url or "invalid" in url` — fragile guard that depends on `conftest.py`'s `unit-test.invalid` choice.
5. `tests/unit/test_admin_config_routes.py:29-34` neutralises `RedisCacheRepository` methods directly on the class at module-import time. Persists across the pytest session — risks cross-test contamination under `pytest-xdist`.
6. `tests/integration/test_data_availability.py` depends on ≥2,000 `odds_api_player_mapping` rows and silently skips when missing.
7. `tests/flow_tests/test_migrate_league_flow.py:13-16` monkey-patches the migration-service internals; orchestration is covered, real branches are not.
8. The match-calibration unit cluster mocks DB session and `predict_team_mean`. CLAUDE rule (`.claude/rules/testing.md`) forbids DB mocks for *integration* tests; unit tests are fine, but there's a coverage trough between mocked unit and subprocess-based integration.
9. **No coverage threshold enforced.** `pyproject.toml [tool.coverage]` is configured but no CI step runs `pytest --cov` or asserts a fail-under.

## 6. CI signal

Workflows in `.github/workflows/`:

| workflow | trigger | runs `pytest tests/unit/`? | runs `pytest tests/integration/`? | runs `pytest tests/flow_tests/`? |
|---|---|---|---|---|
| `test-pipeline.yml:1` | push/PR to `dev` | no — only `tests/run_test.sh` (WebSocket Docker smoke) | no | no |
| `flow-tests.yml:1` | push/PR to `dev` | no | no | **yes** — `bash tests/flow_tests/run_flow_tests.sh --ci` |
| `run-server.yml:1` | manual | no | no | no |
| `claude.yml:1` | issue-comment | no | no | no |
| `notify-docs.yml` | docs only | no | no | no |

Practical consequences:

- The 73 unit tests added under `tests/unit/` (the bulk of the new coverage) are **not exercised by any workflow on the PR-to-`dev` path**. An import error or assertion failure in `tests/unit/` will not fail CI.
- The 4 integration tests under `tests/integration/` are gated by `--run-integration` (not passed by any workflow). Local-developer-only.
- `tests/test_odds_pricing_flows.py` (1594 lines) lives at `tests/` root and is also not invoked by any workflow.
- `tests/pr102/test_e2e_kafka_betting_flow.py` is a CLI script (no `def test_*`); not in any workflow.
- The green badge confirms only (a) the WebSocket Docker stack still boots, and (b) the flow tests pass under Docker. It does **not** confirm that the 73 new unit tests run, parse, or pass.

## 7. Bottom line

- The PR adds substantial useful tests in absolute terms (~24k LOC of test code with thoughtful structure).
- The match-calibration service (1989 lines) gets the densest coverage across ~15 unit files and 1 integration regression — best-defended area in the PR.
- The cluster models (~2,500 lines added across 5 files) get a single in-tree test (370 lines) focused on cluster-zone distribution math; the predictor's full surface and the fitter's persistence/seed paths are not pinned.
- The Kafka lineup pipeline (~1,870 added LOC) has **zero unit tests**; only `tests/test_odds_pricing_flows.py` exercises a small mocked slice.
- **The single most important fix-before-merge: the 73 new unit tests under `tests/unit/` are not invoked by any CI workflow.**

## 8. Top recommendations

(From sub-report C.6.)

1. **Gate the new unit suite in CI.** Add a minimal step to a workflow on the PR-to-`dev` path:
   ```
   - run: PYTHONPATH=src pytest tests/unit -m "not integration and not live and not e2e"
   ```
   Add `tests/test_odds_pricing_flows.py` to that invocation or move it under `tests/unit/`.
2. **Add a focused unit test for `scheduled_odds_fetch.run_scheduled_odds_fetch`** — empty-payload skip, retry on transient Kafka error, poison-message log + continue, heartbeat write.
3. **Add unit tests for `KafkaLineupConsumer` / `KafkaLineupProducer`** — round-trip a representative event, assert the `is_starter` filter is honoured, assert message-format drift surfaces as an exception (not a silent drop).
4. **Replace the JWT mock in `tests/unit/test_admin_config_routes.py:32-34` with a real round-trip against `jwt_helper.py`** — sign, verify, expiry. Or add `tests/unit/test_jwt_helper.py`.
