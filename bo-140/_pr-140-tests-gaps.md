---
title: "BO #140 — Sub-report C: prod files with no test coverage"
pr: TBG-AI/Backend-Odds#140
scope: every non-test src/**/*.py file the PR adds or modifies
audited_at: 2026-04-29
methodology: |
  For each changed src .py file, grep tests/ + src/backend_odds/core/prediction_models/tests
  for any of: `from <full.dotted.path>`, `import <full.dotted.path>`,
  or `from <parent.pkg> import <module_basename>`. Excludes archive dirs
  and string-only mentions (docstrings, comments). Caveats listed below.
---

# Sub-report C — Coverage gaps

## C.1 Method

The PR changes 96 unique `.py` files under `src/`. For each one this audit
counts the number of test files (under `tests/` and the in-tree
`src/backend_odds/core/prediction_models/tests/`) that import the module
via a recognised import statement. The full coverage map is in
`/tmp/pr140-final-coverage.tsv`. Distribution:

| importer count | # of changed src files |
|---|---|
| 0 | 44 (42 prod modules + 2 test modules) |
| 1 | 30 |
| 2 | 10 |
| 3 | 5 |
| 4 | 1 |
| 6 | 1 |
| 7 | 2 |
| 8 | 1 |
| 13 | 2 |

### Caveats (false positives in the "0 importers" bucket)

- **Dynamic imports** are missed. Example: `tests/unit/test_admin_config_routes.py:83-84`
  uses `importlib.import_module("backend_odds.infrastructure.api.rest.routers.admin")`,
  so `routers/admin.py` shows 0 importers. This is real but not a coverage gap.
- **`importlib.util.spec_from_file_location`** is used by
  `tests/unit/scripts/test_verify_pricing_pipeline.py:34` and a few siblings;
  same caveat.
- The two test files in the 0-bucket (`src/backend_odds/core/prediction_models/tests/test_zonal_model.py:1`,
  `src/backend_odds/core/prediction_models/tests/test_cluster_zone_distributions.py:1`)
  are themselves leaf tests that don't get imported by anything — that's
  expected, not a gap.

So **42 production .py files have no detectable test importer**, of which
~3-5 may be false positives due to dynamic import patterns. The list below
flags each entry.

## C.2 High-impact gaps (>50 added LOC, no test importer)

Every entry below is a production module the PR changes substantially with
no test that imports it (per the method in C.1). Annotated with status
and add/del LOC counts from the GitHub API. "0/0" entries indicate the
PR diff was too large for the API to inline; LOC on disk is given in the
notes column.

| status | added/deleted | file (`file:line`) | size on disk | notes |
|---|---|---|---|---|
| modified | +88 / −63 | `src/backend_odds/core/data/data_access/mom_repository.py:1` | 601 lines | MoM repository rewrite. Drives MoM markets (user-facing odds). No test imports it. |
| modified | +74 / 0 | `src/backend_odds/core/data/data_access/pipeline_config_repository.py:1` | 252 lines | All pipeline config reads route through this. No test imports it. |
| modified | +64 / −8 | `src/backend_odds/core/data/data_process/match_statistics.py:1` | n/a | New post-process branches and SF-only fast paths. `tests/unit/migrations/test_sf_match_statistics_mapping.py:1` covers mapping but does not import this module by name; behaviour-on-modified-rows is not pinned. |
| modified | +40 / −2 | `src/backend_odds/infrastructure/api/rest/routers/odds.py:1` | n/a | Router wiring. No test imports it. |
| modified | +28 / −4 | `src/backend_odds/infrastructure/api/rest/routes.py:1` | n/a | Aggregator. Untested. |
| modified | +25 / −5 | `src/backend_odds/core/data/data_process/player_statistics.py:1` | n/a | Untested. |
| modified | +22 / −4 | `src/backend_odds/infrastructure/dependencies/ws_dependencies_copula.py:1` | n/a | Copula dependency wiring. Untested. |
| modified | +18 / −5 | `src/backend_odds/infrastructure/repositories/live_match/match_lineups_repository.py:1` | n/a | Untested. |
| modified | +13 / −4 | `src/backend_odds/core/data/data_access/lineups_repository.py:1` | 232 lines | Untested. |
| modified | +112 / −5 | `src/backend_odds/infrastructure/web/app.py:1` | n/a | New startup wiring (`odds_request_producer.start()`, `run_scheduled_odds_fetch(...)`). No test asserts these run or are gated by a flag. |
| modified | +11 / 0 | `src/backend_odds/infrastructure/cache/cache_utils.py:1` | 332 lines | The `RedisCacheRepository.mget` cluster-safety fix called out in `CLAUDE.md` lives in `redis_repo.py` (covered indirectly via `test_admin_config_routes.py`); this `cache_utils.py` companion has 11 added lines and no test importing the module. |
| added | +350 / 0 | `src/backend_odds/infrastructure/kafka/scheduled_odds_fetch.py:1` | n/a | The scheduled-fetch loop. Wired in via `web/app.py`. The integration test `tests/integration/test_data_availability.py:1` checks the **outputs** (`live_odds:{gid}`, `player_props:{gid}`) but never imports the module or asserts the loop's retry / backoff / poison-message handling. |
| added | +297 / 0 | `src/backend_odds/core/data/data_access/lineup_training_repository.py:1` | n/a | Lineup training data access. Imports `lineup_models.params.PlayerFeatures` and `lineup_models.schemas`. No test imports it. The `is_starter` filter the starter-only assumption depends on is here. |
| added | +268 / 0 | `src/backend_odds/application/service/info/migration_service.py:1` | n/a | Driver for league migrations. The flow test `tests/flow_tests/test_migrate_league_flow.py:13-16` **monkey-patches** the migration service internals — so the orchestrator is exercised but the service's own branches (unsupported league, partial-run resumption, idempotency math) are not. No unit test imports the module. |
| added | +267 / 0 | `src/backend_odds/infrastructure/kafka/lineup_consumer.py:1` | n/a | `KafkaLineupConsumer` class. No test imports it. Wired in via `infrastructure/api/tasks.py`. |
| added | +243 / 0 | `src/backend_odds/core/prediction_models/cluster_models/types.py:1` | n/a | Cluster model TypedDicts/Enums. The other cluster-models files (`fitter.py`, `fitted_model.py`, `predictor.py`) each have 1 importer — `src/backend_odds/core/prediction_models/tests/test_cluster_zone_distributions.py:1` (370 lines, distribution-math focus) — but `types.py` itself has no importer. |
| added | +202 / 0 | `src/backend_odds/application/service/backtest/_backtest_scenarios.py:1` | n/a | Back-test scenario definitions. `tests/unit/application/test_backtest_runner.py:62-70` patches `_build_pipeline_config` to avoid importing this module, so the scenario builder itself is not exercised. |
| added | +159 / 0 | `src/backend_odds/application/service/lineup/lineup_change_detector.py:1` | n/a | Decides when to re-price on lineup change. No test imports it. Quiet failure here = stale prices or cache thrash. |
| added | +123 / 0 | `src/backend_odds/infrastructure/kafka/lineup_producer.py:1` | n/a | `KafkaLineupProducer` class. No test imports it. |
| added | +99 / 0 | `src/backend_odds/core/data/data_access/lineup_source_priority.py:1` | n/a | Source-priority lookup. `tests/unit/test_source_selection_config.py:1` covers selection logic but does not import this module by name. |
| added | +97 / 0 | `src/backend_odds/infrastructure/workers/lineup_scraper.py:1` | n/a | Worker entry point. Untested. |
| added | +80 / 0 | `src/backend_odds/core/prediction_models/lineup_models/constants.py:1` | n/a | Lineup-model constants. Untested. |
| added | 0/0 (truncated) | `src/backend_odds/core/prediction_models/lineup_models/fitted_model.py:1` | 415 lines | Untested. |
| added | 0/0 (truncated) | `src/backend_odds/core/prediction_models/lineup_models/schemas.py:1` | 233 lines | Untested. |
| added | 0/0 (truncated) | `src/backend_odds/core/prediction_models/lineup_models/params.py:1` | 101 lines | Untested. |
| added | 0/0 (truncated) | `src/backend_odds/infrastructure/api/rest/routers/data_health.py:1` | 111 lines | Untested router. |
| added | 0/0 (truncated) | `src/backend_odds/infrastructure/api/rest/routers/auth.py:1` | 79 lines | Untested router (auth). The cross-service e2e script `tests/pr102/test_e2e_kafka_betting_flow.py:1` exercises this via HTTP at runtime, but pytest collects 0 tests from that file. |
| added | 0/0 (truncated) | `src/backend_odds/core/utils/jwt_helper.py:1` | 95 lines | The admin-route test (`tests/unit/test_admin_config_routes.py:32-34`) **mocks** the helper out — so no test pins the actual JWT round-trip (sign, verify, expiry). Sensitive given it's the trust root for admin routes. |
| added | 0/0 (truncated) | `src/backend_odds/core/utils/datetime_utils.py:1` | 62 lines | Untested. |

## C.3 Lower-impact gaps (small modifications, no test importer)

These are smaller PR touches but still no test imports them. Risk is bounded;
listing for completeness.

| status | added/deleted | file | notes |
|---|---|---|---|
| modified | +8 / −8 | `src/backend_odds/core/data/data_access/db_factory.py:1` | DB factory tweak. Untested. |
| modified | +4 / −2 | `src/backend_odds/core/eval/backtest.py:1` | Untested. |
| modified | +4 / −2 | `src/backend_odds/core/model_calibration/calibration_script.py:1` | Untested. |
| modified | +3 / 0 | `src/backend_odds/infrastructure/web/middleware/request_context.py:1` | The `ROUTE_TO_FLOW` map (the file CLAUDE.md calls out as the source of route→flow mapping). 3 lines added (probably one new route entry). No test asserts that newly-added routes are in the map; flow-drift detection lives outside this file. |
| modified | +2 / −2 | `src/backend_odds/infrastructure/api/rest/routers/info.py:1` | Untested. |
| modified | +2 / −2 | `src/backend_odds/infrastructure/dependencies/dependencies.py:1` | Untested. |
| modified | +1 / −1 | `src/backend_odds/infrastructure/api/websocket/manager.py:1` | 1-line change. WebSocket manager — CLAUDE.md flags as critical file. |
| modified | +1 / −1 | `src/backend_odds/infrastructure/api/websocket/manager_typed.py:1` | 1-line change. |
| modified | +1 / −1 | `src/backend_odds/infrastructure/api/websocket/handler.py:1` | 1-line change. |
| modified | +1 / −1 | `src/backend_odds/infrastructure/api/rest/routers/parameters.py:1` | 1-line change. |
| modified | 0/0 (truncated) | `src/backend_odds/infrastructure/api/rest/routers/builder.py:1` | Untested as importer; covered indirectly through other tests. |
| modified | 0/0 (truncated) | `src/backend_odds/infrastructure/api/rest/routers/admin.py:1` | **False positive** — dynamically imported by `tests/unit/test_admin_config_routes.py:83-84` via `importlib.import_module(...)`. |
| modified | 0/0 (truncated) | `src/backend_odds/core/prediction_models/mom_models/mom_rating_pipeline.py:1` | Imported indirectly via `cluster_models.feature_extractor`; not directly imported by any test. |

## C.4 Files with thin coverage (1 importer)

These have at least one test importing them. Calling out the ones where
the lone importer is a far-removed test (not a focused unit), since the
boundary case is "tested in passing, not as the system under test":

| file | sole importer | gap |
|---|---|---|
| `src/backend_odds/application/service/betting/exposure_service.py:1` (210 added) | `tests/test_odds_pricing_flows.py:1330` | Imported but exercised only as part of the W5 lineup-model flow; specific exposure-cap math, edge cases, and partial-fill behaviour are not isolated. |
| `src/backend_odds/application/service/info/request_analytics_service.py:1` (255 added) | `tests/test_odds_pricing_flows.py:1485` | Same shape — imported but not exercised as a focused unit. Consumer-side aggregation, Redis counter increment math, and rollup keys are not pinned. |
| `src/backend_odds/infrastructure/api/tasks.py:1` (+347 / −3) | `tests/test_player_props_calibration.py` | `tests/unit/test_assign_and_fit_saga.py:1` covers the assign-and-fit saga, but the new scheduled-task wiring (lineup producer start/stop, scheduled odds fetch tick, calibration scheduling glue) does not have a focused unit test. |
| `src/backend_odds/infrastructure/api/rest/routers/pipeline_admin.py:1` (+448 / −1) | `tests/qa/player_calibration/test_player_calibration.py` | The 448-line addition to this router is exercised only by one flow test (`test_sf_coverage_flows.py`, sub-report B.2) for a few endpoints. Most negative paths (auth failure, validation failures, partial post-process failures) have no unit test. |
| `src/backend_odds/infrastructure/api/rest/routers/live.py:1` (+2 / −2) | `tests/test_kafka_batch_cap_and_is_live_batch.py` | Trivial change, but the live router's new behaviour is touched only by an existing test, not a new dedicated one. |
| `src/backend_odds/core/prediction_models/cluster_models/fitter.py:1` (698 added) | `src/backend_odds/core/prediction_models/tests/test_cluster_zone_distributions.py:1` | 370-line test file focuses on per-cluster zone distribution math (Change 1) and cluster-zonal joint prediction (Change 2). The fitter has many other surfaces (cluster sizing, k-means seed, feature normalisation, persistence) that are not pinned. |
| `src/backend_odds/core/prediction_models/cluster_models/predictor.py:1` (717 added) | same | Same — predictor's full surface (rate prediction, blend weights, fallback when cluster missing) is not exhaustively pinned. |
| `src/backend_odds/core/prediction_models/cluster_models/fitted_model.py:1` (282 added) | same | Pickle / JSON persistence side and cluster-id stability are not asserted. |
| `src/backend_odds/core/data/data_access/games_repository.py:1` (+71 / −2) | `tests/unit/core/data/data_access/test_games_repository.py:1` | Single dedicated unit — sufficient for the size of the change. |
| `src/backend_odds/core/data_scraping/period.py:1` (303 added) | `tests/unit/data_scraping/test_period_sf_empirical.py:1` | Empirical SF strings only; the migrator flow uses the same module via `tests/unit/migrations/test_sf_migrator_period.py:1` (374 lines, but my regex caught it via `migrations/`, not `period.py`). Coverage is reasonable. |

## C.5 Gap summary

- **42 production modules changed by this PR have no test that imports them**, of which 14 added more than 50 LOC.
- The single biggest cluster of zero-coverage code is the **lineup-models / lineup-Kafka pipeline**: `lineup_consumer.py` (267) + `lineup_producer.py` (123) + `lineup_scraper.py` (97) + `lineup_change_detector.py` (159) + `lineup_training_repository.py` (297) + `lineup_source_priority.py` (99) + `lineup_models/{constants,fitted_model,schemas,params}.py` (~830 on disk) ≈ **~1,870 LOC** of new code with no test importer.
- The **scheduled-fetch + analytics pipeline**: `scheduled_odds_fetch.py` (350) + `migration_service.py` (268) + `request_analytics_service.py` (255, 1 distant importer) + `exposure_service.py` (210, 1 distant importer) + `app.py` startup wiring (112). Roughly **~1,200 LOC** with no focused tests.
- **Trust-root JWT helpers** (`jwt_helper.py`, 95 LOC) is mocked out where it could be tested. Important to fix.
- The lone false positive worth dismissing: `routers/admin.py` (0/0 in the diff API, dynamically imported via `importlib.import_module` in the admin-config-routes test).

## C.6 Top three "fix-before-merge" recommendations purely from a gap perspective

1. **Add a focused unit test for `scheduled_odds_fetch.run_scheduled_odds_fetch`** — at minimum: empty-payload skip, retry on transient Kafka error, poison-message log + continue, heartbeat write. This is the only producer of `live_odds:{gid}` and `player_props:{gid}`; if it dies silently, calibration goes stale (and the integration test `test_data_availability.py:1` only checks output, not producer health).
2. **Add unit tests for `KafkaLineupConsumer` / `KafkaLineupProducer`** — round-trip a representative event, assert the `is_starter` filter is honoured, and assert message-format drift surfaces as an exception (not a silent drop). Lineup events drive the starter-only assumption (CLAUDE.md mandatory rule); silent corruption here corrupts every priced ZAP.
3. **Replace the JWT mock in `tests/unit/test_admin_config_routes.py:32-34` with a real round-trip against `jwt_helper.py`** — sign + verify + expiry edge cases. Or add a separate `tests/unit/test_jwt_helper.py`. The current arrangement leaves the trust root unverified.
