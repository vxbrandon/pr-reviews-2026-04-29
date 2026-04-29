---
title: "BO #140 — Sub-report B: integration / flow / api tests"
pr: TBG-AI/Backend-Odds#140
scope: tests/integration/, tests/flow_tests/, tests/api/, tests/pr102/, tests/qa/, tests/test_*.py at repo root
audited_at: 2026-04-29
---

# Sub-report B — Integration, flow, e2e tests

Read-only mapping. All file:line citations target the working tree at
`/Users/tbg/Desktop/programming/Dev-Agentic/Backend-Odds` on the PR branch.

## Tally

- Added under `tests/integration/`: 4 test files + 1 conftest.
- Added under `tests/flow_tests/`: 3 test files (no `tests/api/` files added or modified).
- Added under `tests/pr102/`: 1 file (script — see B.4).
- Added at repo root `tests/`: 1 file (`tests/test_odds_pricing_flows.py`).
- Modified: `tests/test_market_odds_service_live_kalshi.py` (1/1 trivial), `tests/qa/player_calibration/test_player_calibration.py` (1/1 trivial).
- Fixtures added: `tests/fixtures/odds_api/upcoming_events.json` (18 lines, used by sub-report A's `test_market_odds_api_service.py`).

## B.1 Integration tests (`tests/integration/` — gated by `--run-integration`)

All four files share the gating pattern in `tests/integration/conftest.py:30-49`:
the marker `integration` is registered in `pyproject.toml`; `pytest_collection_modifyitems`
auto-skips integration tests unless `--run-integration` is passed. **No CI workflow
in `.github/workflows/` passes that flag**, so these tests are local-developer-only.

| test file | tests added | code under test (`file:line`) | type | gating |
|---|---|---|---|---|
| `tests/integration/conftest.py:1` (49 lines) | registers `--run-integration`; sets `DATABASE_URL` to `unit-test.invalid` for collection-time hygiene | n/a | infra | always |
| `tests/integration/test_calibration_regression.py:1` (692 lines) | URA-T34 regression: spawns a real FastAPI app on port 11002 (`subprocess.Popen`, line 237–273) → runs uncal vs cal pricing → asserts `\|p_cal − p_uncal\| ≥ 0.05` at shots lines 0.5 and 2.5 → asserts `:adj=none` and `:adj=<fingerprint>` cache keys coexist under `odds_cache:zap_odds_*` → identity-adjustment test asserts near no-op within `IDENTITY_TOLERANCE = 0.03` | `application/service/calibration/match_calibration_service.py` (1989 added), `core/prediction_models/orchestration/main_pipeline.py:get_prob_for_zap`, ZAP cache layers (`odds_cache:zap_odds_*`, `batch_zap:*:sim_v5`, `builder:all_zap:*`) | integration (real Postgres + real Redis + real subprocess'd FastAPI) | requires `DATABASE_URL`, `localhost:6379`, fixture game `371402987` |
| `tests/integration/test_data_availability.py:1` (573 lines) | URA-T36: read-only probes for the four calibration data sources — `odds_api_player_mapping` table size & freshness; `player_props:{gid}` Redis key shape & freshness; `live_odds:{gid}` Redis key; `scheduled_odds_fetch:last_run` heartbeat (skipped if absent) | `match_calibration_service._load_*`, `infrastructure/kafka/scheduled_odds_fetch.py:1` (350 added), the URA-T26 player-mapping backfill | integration (real Postgres + real Redis, read-only) | skip if `DATABASE_URL` unset, Redis unreachable, or table missing |
| `tests/integration/test_data_completeness.py:1` (272 lines) | seeds a fully-healthy game (IDs in 99,991+ range, line 32–38) → asserts `DataCompletenessService` returns `alerts == []` → cleanup on teardown | `application/service/info/data_completeness_service.py:1` (338 added) | integration (real Postgres seed + cleanup) | skip if DB unset/unreachable; idempotent across re-runs |
| `tests/integration/test_model_quality.py:1` (778 lines) | URA-T35: rate-tracks base-model / market disagreement on game 371402987 — clamp saturation %, per-player share band (2-30%), team-mean finiteness, factor-direction inversion ≤10%; in-process pipeline (no subprocess) | base models + `match_calibration_service.calibrate_match` | integration (real Postgres + Redis + cached models) | skip on missing fixture, unreachable infra, or cluster model load failure |

## B.2 Flow tests (`tests/flow_tests/` — gated by Docker compose)

Run via `bash tests/flow_tests/run_flow_tests.sh --ci` (line 99 of the script
runs `python -m pytest tests/flow_tests/ -v --tb=short --timeout=120 -x`).
The `--ci` mode brings up `docker-compose.yml` + `docker-compose.ci.yml`, waits
for the Backend-Odds health endpoint, then runs the full flow-tests directory.
The CI workflow `.github/workflows/flow-tests.yml:68` invokes this with `--ci`.

| test file | tests added | code under test (`file:line`) | type |
|---|---|---|---|
| `tests/flow_tests/test_sf_coverage_flows.py:1` (286 lines) | `GET /admin/sources/actions` (200 + dict, WhoScored has flagship actions, SF is narrow subset); `PUT /admin/pipelines/{name}/config` accepts `event_source_preference`; `POST /admin/pipelines/{name}/post-process` 404 / 400 / 200 shapes; `PlayerStatistics` upsert across the 2→3 column constraint rebuild from migration q2r3s4t5u6v7 | `infrastructure/api/rest/routers/pipeline_admin.py:1` (+448/-1) + `core/data/data_process/match_statistics.py` (+64/-8) | flow (HTTP against running Docker app) |
| `tests/flow_tests/test_migrate_league_flow.py:1` (300 lines) | `POST /admin/migrate/{league_code}`: WhoScored runs before SofaScore (ordering contract); both write to `migration_runs` audit; 24h idempotency; `force=true` bypasses; validation rejects empty `league_code` / both-disabled. **Monkey-patches the migration service internals** (line 13–16 docstring acknowledges this) | `application/service/info/migration_service.py:1` (268 added), `migration_runs` audit table | flow |
| `tests/flow_tests/test_season_map_invalidation.py:1` (113 lines) | reproduces 422 "soccer_zones on NoneType" root cause: in-memory `SEASON_TO_GROUP` invalidates on admin writes; `POST /admin/reload-season-map` is operator escape hatch | season-map plumbing in admin router, `pipeline_season_assignments` table | flow |

## B.3 Mixed-location tests added to `tests/`

| test file | tests added | code under test (`file:line`) | type |
|---|---|---|---|
| `tests/test_odds_pricing_flows.py:1` (1594 lines) | covers W1 / W3 / W5 of the odds-pricing improvements workstream — W1: `OddsRequestEvent` JSON round-trip + auto-`timestamp_ms` + `OddsRequestProducer.emit` fire-and-forget when disabled / when underlying producer is `None`. W3: scheduled odds fetch (mocked Kafka). W5: lineup-model integration | `infrastructure/kafka/request_producer.py:1` (174 added), `infrastructure/kafka/scheduled_odds_fetch.py:1` (350 added, only mocked-slice covered), `core/prediction_models/lineup_models/*` | unit-style despite `tests/` location (no marker, no DB/Redis touch — every external is mocked) |

## B.4 Modified tests

| test file | change | notes |
|---|---|---|
| `tests/test_market_odds_service_live_kalshi.py:16` | 1 line modified (1/1) | trivial import-path / rename touch dragged along by the refactor; no new behaviour pinned |
| `tests/qa/player_calibration/test_player_calibration.py:481` | 1 line modified (1/1, in `class TestRefitManager`) | trivial touch; the class name suggests it covers `RefitManager` (`application/service/calibration/refit_manager.py`, +16/-1 in this PR) |

## B.5 e2e — `tests/pr102/test_e2e_kafka_betting_flow.py`

`tests/pr102/test_e2e_kafka_betting_flow.py:1` (399 lines) is **named like a
pytest file but has no `def test_*`** — pytest will collect 0 items from it.
The file ends with `if __name__ == "__main__": main()` (line 398) and uses
`sys.exit(1)` from helper functions on failure. It is a CLI script that
covers:

1. Service health (Backend-Odds 8080, Backend-Server 8000, Redpanda 9092, Redis 6379, Postgres 5432).
2. Admin login on Backend-Server with a date-derived password (line 119–146).
3. Test-user creation via `/admin/generate_test_users` (line 127–134).
4. Bet placement on a discovered game.
5. Verification of Kafka topics `odds.odds_requests`, `odds.user_bets`, `odds.market_updates`, `odds.lineup_updates`.
6. Redis analytics counter verification.

| test file | what it does | type | gating |
|---|---|---|---|
| `tests/pr102/test_e2e_kafka_betting_flow.py:1` (399 lines) | imperative cross-service e2e — auth → bet → Kafka topic checks → Redis analytics | **script, NOT pytest** | Backend-Odds + Backend-Server + Redpanda + Redis + Postgres all running locally |

Code under test for this script (covered nowhere else in this PR):
- `infrastructure/kafka/request_producer.py:1` (event publication path).
- `application/service/info/request_analytics_service.py:1` (255 added) — Redis counter writer.
- The `auth.py` router (added) — admin login flow.
- Cross-repo bet placement → Kafka pipeline.

Status: this file is documentation / manual QA. It is not in any CI workflow.
Either rename to `scripts/manual_qa/` or convert to parametrised pytest cases
under `@pytest.mark.e2e` and add to a new workflow.

## B.6 What CI actually runs

Workflows in `.github/workflows/`:

| workflow | trigger | python tests run? | unit suite run? | integration suite run? | flow suite run? |
|---|---|---|---|---|---|
| `test-pipeline.yml:1` | push/PR to `dev` | only the WebSocket Docker smoke (`./tests/run_test.sh`) | no | no | no |
| `flow-tests.yml:1` | push/PR to `dev` + workflow_dispatch | yes (via `bash tests/flow_tests/run_flow_tests.sh --ci`) | no | no | **yes — `tests/flow_tests/`** |
| `run-server.yml:1` | workflow_dispatch only | no | no | no | no |
| `claude.yml:1` | issue-comment trigger only | no | no | no | no |
| `notify-docs.yml` | docs only | no | no | no | no |

Conclusion for sub-report B: of the 9 integration / flow / e2e files added by
this PR, **3 run in CI** (the three flow tests gated by `flow-tests.yml`),
**4 are local-developer-only** (the integration tests requiring `--run-integration`),
**1 is a non-pytest script** (`tests/pr102/test_e2e_kafka_betting_flow.py`),
and **1 is unit-style despite its location** (`tests/test_odds_pricing_flows.py`,
which would also run in any sensible `pytest tests/` invocation but is not
invoked anywhere in the workflow set).

## B.7 Quality observations

1. `tests/integration/test_calibration_regression.py:237-273` spawns a child Python process via `subprocess.Popen(... preexec_fn=os.setsid ...)` on port 11002. Cleanup is `os.killpg(os.getpgid(...))`. There is no `try/finally` around the launch in the obvious place — if a non-pytest exception fires before the kill path, the child can leak. The 32-second `SLEEP_AFTER_ADJ_CHANGE_SEC` (line 101) means each pricing-cycle assertion takes ~32s; total runtime is ~2 minutes. CI flake risk on slow runners.
2. `tests/integration/test_calibration_regression.py:134` and `tests/integration/test_data_completeness.py:43` skip on `if not url or "invalid" in url`. Works because `conftest.py` sets `unit-test.invalid` deliberately, but a future fixture host containing `invalid` would silently skip. Tighter: gate on `DATABASE_URL_INTEGRATION` or fail-loud when `--run-integration` is on but the env is misconfigured.
3. `tests/integration/test_data_availability.py` is read-only and depends on `odds_api_player_mapping` having ≥ 2,000 rows (line 80). Fresh local DBs without URA-T26 backfill silently skip — no surfaced "I wanted to run but couldn't" signal.
4. `tests/flow_tests/test_migrate_league_flow.py:13-16` monkey-patches the migration service internals — the flow test asserts orchestration ordering, not the actual data movement. Coverage of `MigrationService` real branches (unsupported league, partial-run resumption, idempotency math) lives nowhere in this PR.
5. `tests/test_odds_pricing_flows.py` lives at the `tests/` root (not under `tests/unit/`) and therefore won't be picked up by a future `pytest tests/unit/` invocation either. If the recommendation in sub-report A.7 / sub-report C is to add `pytest tests/unit/` to CI, this file would be missed; it should move under `tests/unit/` (or `tests/integration/` if reclassified).
6. `tests/flow_tests/run_flow_tests.sh:103` uses `-x` (stop on first failure). For CI signal that means "first failure hides all other failures" — fine for fast feedback locally, less ideal as a merge gate where you want the full failure picture.
7. The two trivial test modifications (B.4) are import-path / rename fix-ups, not new coverage. They should not be counted as "tests added/modified" when reasoning about coverage.
