# PR #140 — Architecture & Service Surface Audit

**PR**: TBG-AI/Backend-Odds#140 ("Dev coverage and odds impro")
**Base / Head**: `dev` ← `dev-coverage-and-odds-impro`
**Scale**: 330 changed files, +88,029 / −2,160
**Scope of this audit**: services under `src/backend_odds/application/service/`, repositories under `src/backend_odds/core/data/data_access/`, Alembic migrations (`alembic/versions/` + `alembic_ext_odds/`), DTOs/schemas crossing service boundaries, configuration surface, and notable architectural decisions. Read-only; line citations are at-HEAD on the PR branch.

---

## 1. Service surface

The PR introduces a flat → grouped reorganisation of `application/service/`. Six subfolders now exist: `odds/`, `calibration/`, `backtest/`, `betting/`, `lineup/`, `info/`. Renames (no content change) preserve git history; new files carry the bulk of the W1–W5 functionality.

### 1a. New service modules

| Service | Purpose (1 line) | File:line | Key public API |
|---|---|---|---|
| `BacktestRunner` | Single harness collapsing the duplicated temporal-cutoff loops in `scripts/odds_qa_testing/backtest_cluster_model.py`, `scripts/eval/run_eval.py`, `scripts/debug/cluster/evaluate_cluster_model.py`. Pluggable `BacktestScenario` enum. | `application/service/backtest/backtest_runner.py:234` | `BacktestRunner.run(request: BacktestRequest) -> BacktestResult`; `BacktestScenario` (enum, line 67); `BacktestRequest`/`BacktestResult` dataclasses (lines 91, 115); `BacktestLeakageError` (line 137); `_assert_no_leakage()` (line 167) |
| `_backtest_scenarios` | Scenario→config translator, split out purely to keep the runner under the 600-LOC contract. Mutual import with the runner. | `application/service/backtest/_backtest_scenarios.py:25` | `_ScenarioMixin`; `build_pipeline_config()` (32); `position_lookup_for_scenario()` (72); `mae`/`ece`/`roi` metric helpers (109/119/141); `coerce_to_utc` (153); `list_games`/`iter_stat_rows` (170/189) |
| `ExposureService` | Reads per-market / per-match bet exposure counters from Redis hashes (`exposure:{market_key}`) and persists snapshots into the `exposure_snapshots` table. Pairs with the existing `BetConsumer` in tbg-streaming. | `application/service/betting/exposure_service.py:52` | `MarketExposure` dataclass (39); `ExposureService.snapshot_market(...)`, `.snapshot_all(...)` |
| `MatchCalibrationService` (W2) | Per-match multiplicative calibration. Computes lightweight adjustment factors so model-implied probabilities track external books for moneyline, player props, and team props. The single largest new module in the PR (~2,000 LOC). | `application/service/calibration/match_calibration_service.py:524` | `calibrate_match(game_id, ...)` (549); `get_adjustment(game_id)` (753); `MatchCalibrationAdjustment` dataclass with `to_json/from_json` (455, 489, 493); plus `is_match_calibration_enabled()` module-level (56). Internals: `_optimise_factors` (813), `_calibrate_player_props` (1119), `_calibrate_team_props` (1617) |
| `DataCompletenessService` | Read-only completeness diagnostics over `games.data_locked` plus the per-source coverage columns added in PR #109 (`has_whoscored_events`, `has_sofascore_events`, `*_event_count`). Drives `/admin/data-health`. | `application/service/info/data_completeness_service.py:70` | `GameCompletenessReport` (49); `DataCompletenessService.get_report(game_id)`, `.evaluate_alerts()`; module-level `_evaluate_alerts` (245) |
| `MigrationService` | Orchestrates the canonical "WhoScored → SofaScore" migration pair per league, persists per-step audit rows in `migration_runs`, and exposes a single async `migrate_league()` entry point. | `application/service/info/migration_service.py:75` | `StepResult` (34); `MigrationResult` (56); `MigrationService.migrate_league(league_code, force=False)` |
| `RequestAnalyticsService` | Background-thread Kafka consumer (`odds.odds_requests` topic) that maintains lightweight Redis counters for odds-request telemetry. | `application/service/info/request_analytics_service.py:54` | `RequestAnalyticsService.start()`, `.stop()`, plus accessor methods for counter/window reads |
| `LineupChangeDetector` (W4) | Diffs a freshly-scraped `LineupUpdate` against the previously-known lineup in Redis; emits a Kafka event only on real change. | `application/service/lineup/lineup_change_detector.py:66` | `LineupDiff` (35); `LineupChangeDetector.detect(update) -> LineupDiff \| None` |
| `MarketOddsApiService` | Single deep service for `the-odds-api.com`. Replaces auth/session/dedup/upsert duplication across `fetch_player_prop_odds.py`, `fetch_all_market_odds.py`, `fetch_single_event_odds.py`, `fetch_afcon_h2h_odds.py`, `fetch_kalshi_odds.py`. | `application/service/odds/market_odds_api_service.py:83` | `fetch_upcoming_events(league)` (129); `fetch_h2h_totals_spreads(event_id, markets)` (149); `fetch_player_props(event_id, props)` (169); `fetch_historical_at(event_id, ts)` (196); `persist(snapshot, db_session)` (217); domain errors `OddsApiAuthError`/`RateLimitError`/`ResponseError` (71/75/79) |
| `extreme_factor_alerts` | Module-level helpers backing the `passthrough_with_alert` policy on `MATCH_CALIBRATION_EXTREME_POLICY` (URA-T41). Pushes structured events to a Redis sorted set for human review. | `application/service/odds/extreme_factor_alerts.py` | `record_extreme_factor(...)` (43); `drain_alerts(...)` (107) |
| `vig_policy` | Pure-math vig/overround application. Centralises three operations previously inlined across pricing paths. No I/O, no pipeline state. | `application/service/odds/vig_policy.py` | `apply_divisive_vig` (38) — single-outcome (ZAP/ZAT/single-ZAO); `apply_power_vig` (74) — multi-outcome (MOM/ZAO-most/player props); `invert_power_vig` (141) — used by player-props calibration to undo the book's vig |

### 1b. Renamed / relocated services (content largely preserved)

| New path | Old path | Net diff |
|---|---|---|
| `application/service/betting/live_betting_service.py` | `application/service/live_betting_service.py` | +1/−1 |
| `application/service/betting/live_policy.py` | `application/service/live_policy.py` | 0/0 |
| `application/service/calibration/parameter_service.py` | `application/service/parameter_service.py` | 0/0 |
| `application/service/calibration/refit_manager.py` | `application/service/refit_manager.py` | +16/−1 |
| `application/service/info/info_service.py` | `application/service/info_service.py` | 0/0 |
| `application/service/odds/builder_service.py` | `application/service/builder_service.py` | 0/0 (but DTOs grew — see §4) |
| `application/service/odds/live_odds_service.py` | `application/service/live_odds_service.py` | +22/−15 |
| `application/service/odds/market_odds_service.py` | `application/service/market_odds_service.py` | +23/−4 |
| `application/service/odds/odds_service.py` | `application/service/odds_service.py` | +18/−24 |

`GetOddsForBet` (the main `odds_service.py` entry point) at `application/service/odds/odds_service.py:129` retains its public surface (`get_live_odds`, `get_mom_odds`, `get_odds_for_all_players`, `get_odds_for_zao`, `get_live_odds_status`, etc.).

---

## 2. Repositories

| Repository | Purpose | File:line | Key public API |
|---|---|---|---|
| `LineupTrainingRepository` (NEW) | Pulls feature vectors from `lineups` + `games` for `FittedLineupModel.fit()` and `PlayerFeatures` for prediction. | `core/data/data_access/lineup_training_repository.py:38` | `get_training_data(...)` (47); `get_player_features_for_team(...)` (197) |
| `lineup_source_priority` (NEW, helpers) | De-dup helpers for reading `LineupModel` rows once both `whoscored` + `sofascore` providers may insert per game. Returns single-source-priority `Select`/`Subquery`. | `core/data/data_access/lineup_source_priority.py` | `select_preferred_lineups()` (65); `preferred_lineups_subquery()` (85) |
| `event_source` (NEW, helpers) | Per-game source resolution between WhoScored / SofaScore. Constants + `resolve_event_source()`. | `core/data/event_source.py` | `SOURCE_WHOSCORED`, `SOURCE_SOFASCORE`; `MIN_EVENT_COUNT = 5`; `get_source_actions(source)` (42); `resolve_event_source(...)` (51) |
| `GamesRepository` | +71/−2. Adds source-aware queries (`has_whoscored_events`, `has_sofascore_events`, etc.) introduced by `q2r3s4t5u6v7`. | `core/data/data_access/games_repository.py:18` | `class GamesRepository(BaseRepository[Games])` |
| `MatchStatisticsRepository` | +57/−12. Source filtering for new `match_statistics.source` column; pre-existing PK now `(…, source)`. | `core/data/data_access/match_statistics_repository.py:16` | `class MatchStatisticsRepository(BaseRepository[MatchStatistics])` |
| `MOMRepository` | +88/−63. Substantial rework around lineup source de-dup. | `core/data/data_access/mom_repository.py:24` | `class MOMRepository(BaseRepository[LineupModel])` |
| `PipelineConfigRepository` | +74/−0. New helpers for pipeline group configuration / season assignment lookups. | `core/data/data_access/pipeline_config_repository.py:19` | `class PipelineConfigRepository(BaseRepository[PipelineGroup])` |
| `PlayerStatisticsRepository` | +89/−15. Source-aware reads against `player_statistics.source`. | `core/data/data_access/player_statistics_repository.py:36` | `class PlayerStatisticsRepository(BaseRepository[PlayerStatistics])` |
| `PlayersRepository` | +55/−23. | `core/data/data_access/players_repository.py:14` | `class PlayersRepository(BaseRepository[Players])` |
| `ZonalStatisticsRepository` | +24/−4. | `core/data/data_access/zonal_statistics_repository.py:10` | `class ZonalStatisticsRepository(BaseRepository)` |
| `LineupRepository` | +13/−4. Wired through `lineup_source_priority`. | `core/data/data_access/lineups_repository.py:16` | `class LineupRepository(BaseRepository[LineupModel])` |
| `db_factory` | +8/−8. Touch-up (likely engine/session wiring for two alembic histories). | `core/data/data_access/db_factory.py` | factory functions |
| `data_process/match_statistics.py` | +64/−8. Picks up source resolution. | `core/data/data_process/match_statistics.py` | – |
| `data_process/player_statistics.py` | +25/−5. | `core/data/data_process/player_statistics.py` | – |

---

## 3. Migrations

### 3a. `alembic/versions/` (main odds DB)

| Revision | Tables affected | Op | Risk | Reversible? | Notes |
|---|---|---|---|---|---|
| `q2r3s4t5u6v7` add_event_source_tracking | `games`, `events`, `match_statistics`, `player_statistics` | `add_column` ×8, `create_index` ×5, **PK swap** on `match_statistics` & `player_statistics` to include `source` | **HIGH** | Yes (mirrors upgrade with concurrent index drops) | Adds `has_whoscored_events`, `has_sofascore_events`, `whoscored_event_count`, `sofascore_event_count`, `preferred_event_source`, `event_source_resolved` to `games` (`alembic/versions/q2r3s4t5u6v7_add_event_source_tracking.py:44-49`); adds `source` (default `whoscored`) to `events`, `match_statistics`, `player_statistics`. Pre-checks for dupe keys before constraint swap (line 78-91). Uses `CREATE UNIQUE INDEX CONCURRENTLY` inside `op.get_context().autocommit_block()` to avoid `ACCESS EXCLUSIVE` locks; promotes via `ALTER TABLE … ADD CONSTRAINT … USING INDEX`. PK rename via raw `op.execute('ALTER TABLE … DROP CONSTRAINT …')` (line 155-156) is the riskiest step. |
| `62a181c29a0b` merge_event_source_tracking_match_lock_ | — | merge (no-op upgrade/downgrade) | LOW | Yes | Merges heads `h8i9j0k1l2m3` + `q2r3s4t5u6v7`. Empty body. |
| `r3s4t5u6v7w8` add_sf_match_statistics_table | `sf_match_statistics` (create), `match_statistics` (delete WHERE source='sofascore') | `create_table`, `create_index`, `op.execute(DELETE …)` | **MED** | Partially — the DELETE is irreversible in `downgrade()` by design. | New SF-keyed team-stats table mirroring Backend-Server. **One-shot idempotent cleanup** of zombie SF rows in `match_statistics` that carried hardcoded zeros (line 88-100). Downgrade explicitly does *not* restore deleted rows; operators must re-run the SF migrator. |
| `s4t5u6v7w8x9` add_source_to_lineups | `lineups` | `add_column` (source, default `whoscored`), pre-check dupes, swap unique constraint to `(game_id, player_id, source)` | **MED-HIGH** | Yes | Same zero-downtime pattern as `q2r3s4t5u6v7` (concurrent index, `USING INDEX`). Downgrade `DELETE FROM lineups WHERE source = 'sofascore'` then drops column (line 106). |
| `mig_audit_202604230358` add_migration_runs_audit_table | `migration_runs` (create) | `create_table`, `create_index` | LOW | Yes | New audit table for `async_migrate_*.py` scripts: 24h idempotency gate, `--force` override. Cols: `script_name`, `league_code`, `started_at`, `ended_at`, `status` (`running\|succeeded\|failed`), `rows_touched_by_table` (JSONB), `error`, `force_flag`. Index on `(script_name, league_code, started_at DESC)`. |
| `6392e7d01782` merge_migration_runs_audit_and_source_ | — | merge | LOW | Yes | Empty merge of `mig_audit_202604230358` + `s4t5u6v7w8x9`. |

**Three head-merges in one PR** (`62a181c29a0b` and `6392e7d01782`) is a strong signal of parallel branch development. Final head is `6392e7d01782`.

`alembic_soccer_data.ini` (−119 lines, removed) — config file deleted; not replaced.

### 3b. `alembic_ext_odds/versions/` (NEW alembic history)

| Revision | Tables affected | Op | Risk | Reversible? | Notes |
|---|---|---|---|---|---|
| `f0a1b2c3d4e5` initial_ext_odds_schema | 23 tables in **NEW `ext_odds` SCHEMA** | `CREATE SCHEMA IF NOT EXISTS ext_odds`, `create_table` ×23 + ~30 indexes | LOW (greenfield) | Yes (full inverse) | Greenfield baseline for the `ext_odds_streaming` sub-package. Manually constructed (autogenerate forbidden in the overnight loop) — claims to mirror what `alembic revision --autogenerate` would emit with `compare_type=True, compare_server_default=True`. Tables (via `f0a1b2c3d4e5_initial_ext_odds_schema.py:39-925`): `providers`, `fetch_log`, `moneyline_odds`, `totals_odds`, `spread_odds`, `alternate_spreads`, `alternate_totals`, `half_moneyline`, `half_spreads`, `half_totals`, `btts_odds`, `double_chance_odds`, `draw_no_bet_odds`, `team_totals_odds`, `corner_totals_odds`, `corner_spreads_odds`, `card_totals_odds`, `card_spreads_odds`, `player_goalscorer_odds`, `player_shots_odds`, `player_shots_on_target_odds`, `player_assists_odds`, `player_cards_odds`. Schema constant `SCHEMA = "ext_odds"`. |

Both alembic envs gain `compare_type=True, compare_server_default=True` (`alembic/env.py:60-61, 81-82`; `alembic_ext_odds/env.py:54-55, 80-81`) — strict autogenerate fidelity for both online + offline modes.

---

## 4. Schema / DTOs crossing service boundaries

### 4a. `MarketOddsApiService` DTOs (NEW, frozen dataclasses)

`application/service/odds/market_odds_api_dtos.py` — explicitly small, opaque payloads passed back to `MarketOddsApiService.persist()`:

| Type | File:line | Shape |
|---|---|---|
| `Event` | `:17` | event_id, league, kickoff, etc. |
| `MarketOutcome` | `:40` | name + price |
| `MarketSnapshot` | `:52` | event + outcomes, fetched_at |
| `PlayerPropOutcome` | `:71` | player + prop + line + price |
| `PlayerPropSnapshot` | `:88` | event + outcomes |

### 4b. `BuilderService` typed dicts (renamed module, but TypedDict surface is the user-facing contract for the response builder)

`application/service/odds/builder_service.py`:

| TypedDict | File:line | Purpose |
|---|---|---|
| `TeamInfo` | `:19` | nested team payload |
| `AllOddsMetadata` | `:24` | response metadata |
| `AllOddsResponse` | `:32` | top-level shape |
| `ZapItem` | `:42` | per-player ZAP |
| `ZatItem` | `:61` | per-team ZAT |
| `TeamPropItem` | `:77` | team-prop item |
| `ZaoItem` | `:90` | per-zone ZAO |
| `MomItem` | `:100` | MOM item |
| `_AllOddsContext` | `:122` | private builder state |
| `ZoneGroupConfig` (BaseModel) | `:160` | zone-grouping config |

### 4c. `LiveOddsService` typed dicts

`application/service/odds/live_odds_service.py`:

| Type | File:line |
|---|---|
| `VigConfig` | `:30` |
| `MoneylineEntry` (TypedDict) | `:48` |
| `SpreadEntry` (TypedDict) | `:57` |
| `OverUnderEntry` (TypedDict) | `:67` |

### 4d. `MarketOddsService` typed dicts

`application/service/odds/market_odds_service.py`:

| Type | File:line |
|---|---|
| `MarketMoneyline` | `:40` |
| `MarketSpread` | `:47` |
| `MarketTotals` | `:54` |
| `MarketOddsSummary` | `:64` |
| `LiveKalshiFetchEntry` | `:84` |

### 4e. `LiveBetting` schemas

`application/service/betting/live_policy.py` — Pydantic + Enums centralising "is this market currently live-priceable?" contract:

| Type | File:line |
|---|---|
| `LiveOddsStatus` (str Enum) | `:34` |
| `LiveMarketType` (str Enum) | `:62` |
| `IsLiveResponse` (BaseModel) | `:74` |
| `IsLiveBatchRequest` (BaseModel) | `:81` |
| `IsLiveBatchResponse` (BaseModel) | `:92` |

### 4f. Backtest contracts

`application/service/backtest/backtest_runner.py`:

| Type | File:line | Notes |
|---|---|---|
| `BacktestScenario` (Enum) | `:67` | the pluggable axis |
| `BacktestRequest` (dataclass) | `:91` | input contract |
| `BacktestResult` (dataclass) | `:115` | output contract |
| `_PipelineFactory` (Protocol) | `:152` | injectable factory |
| `_RepoFactory` (Protocol) | `:156` | injectable factory |

---

## 5. Config & infra

### 5a. `.required-env` (+33 / −1)

The PR adds **33 new env vars** in 3 logical groups (`.required-env:206-272, 336-340`):

| Group | Vars | Default behaviour |
|---|---|---|
| **Lineup model (W5)** | `LINEUP_MODEL_ENABLED`, `LINEUP_MODEL_MIN_APPEARANCES`, `CLUSTER_MODEL_ENABLED`, `LINEUP_MODEL_CACHE_TTL_SEC`, `LINEUP_CONFIRMED_OVERRIDE` | `[optional]` flags; off by default |
| **Scheduled odds fetch (W3)** | `ODDS_FETCH_ENABLED`, `ODDS_FETCH_INTERVAL_SEC` (default 300), `ODDS_FETCH_LOOKAHEAD_HOURS` (24), `ODDS_FETCH_MARKETS` (`h2h,totals,spreads`) | `[optional]`, off by default |
| **Match calibration (W2)** | `MATCH_CALIBRATION_ENABLED` (default false), `MATCH_CALIBRATION_MAX_DELTA` (0.15 = ±15%), `MATCH_CALIBRATION_CONFIDENCE_FLOOR` (0.3), `MATCH_CALIBRATION_COOLDOWN_SEC` (60), `MATCH_CALIBRATION_TTL_SEC` (1800), `MATCH_CALIBRATION_DIVERGENCE_THRESHOLD` (0.05), `MATCH_CALIBRATION_CHECK_INTERVAL_SEC` (120), `MATCH_CALIBRATION_PLAYER_PROPS_ENABLED` (true), `MATCH_CALIBRATION_PLAYER_PROP_HOLD` (0.15), `MATCH_CALIBRATION_PLAYER_PROP_GAMMA` (0.90), `MATCH_CALIBRATION_PLAYER_PROP_R` (5.0) | `[optional]`, off by default |
| **Lineup scraper (W4)** | `LINEUP_SCRAPER_ENABLED` (false), `LINEUP_SCRAPER_INTERVAL_SEC` (120), `LINEUP_SCRAPER_LOOKAHEAD_HOURS` (24), `KAFKA_LINEUP_TOPIC` (`odds.lineup_updates`), `KAFKA_LINEUP_GROUP` | `[optional]`, off by default |
| **Odds request tracking (W1)** | `ODDS_REQUEST_TRACKING_ENABLED` (default true), `ODDS_REQUEST_KAFKA_TOPIC` (`odds.odds_requests`), `ODDS_REQUEST_ANALYTICS_GROUP` (`backend-odds-request-analytics`) | `[optional]`, **on by default** |

`EXT_ODDS_FETCH_INTERVAL` is now annotated `(legacy)`.

`.env.local` (+6), `.env.stage` (+33), `.env.prod` (+33) — env-template churn matching the new vars.

### 5b. `pyproject.toml` (+8 / −5)

- `scripts/archive` → `scripts/_archive` rename across `black.exclude`, `isort.skip`, `ruff.extend-exclude`, `pylint.ignore`, `coverage.omit` (lines 5, 11, 23-24, 62, 82). New `tests/_archive/` exclusion across the same.
- New pytest marker: `integration: integration tests that spin up the real Backend-Odds app and hit local Redis + Postgres. Opt-in via --run-integration.` (line 53).

### 5c. `.gitignore` (+11 / −1) — not exhaustively read; growth correlates with new artifact dirs.

### 5d. Alembic env hardening

Both `alembic/env.py` and `alembic_ext_odds/env.py` add `compare_type=True, compare_server_default=True` to *both* online and offline modes (cited in §3). Material because all forthcoming autogenerate runs will detect column-type and server-default drift that current heads silently allow.

### 5e. Tooling additions

| Path | Status | Notes |
|---|---|---|
| `AI_DOCS/INDEX.md` | NEW | +29 lines — index of the 19 new reference / research docs in this PR. |
| `AI_DOCS/reference/2026-04-13_W{1..5}-*.md` | NEW | Per-workstream design docs (W1 betting-data-pipeline, W2 match-calibration, W3 scheduled-odds-fetch, W4 lineup-scraping-kafka, W5 lineup-model + revised zone-cluster-model). |
| `AI_DOCS/reference/2026-04-17_audit-{01..06}-*.md`, `audit-all.md/.html`, `backend-odds-audit.md/.html` | NEW | A 6-part architecture audit (migration, modeling, fitting, inefficiency, organization, vs-streaming) — ~28K lines combined. |
| `AI_DOCS/reference/2026-04-17_odds-pricing-exploitability-investigation.md`, `2026-04-18_ralph-audit-fix-report.md` | NEW | Ralph-loop audit-fix output. |
| `AI_DOCS/research/{2025-04-07_player-action-modeling-research.md, CODE_WALKTHROUGH_player-modeling.md, QUICKREF_player-modeling.md, README.md}` | NEW | Research artefacts. |
| `docs/pr/ralph-loop-agents-guide.md` | NEW | Operational AGENTS.md for the audit-fix loop (env, validation gate, "do NOT run `make check`/alembic upgrade head/docker compose"). |
| `docs/CHANGELOG-phase-1.md`, `docs/migrations/README.md`, `docs/modeling/cluster_model_math.md`, `docs/playbooks/fa-cup-validation.md`, `docs/pr/odds-pricing-improvements.md`, `docs/pr/parameter-management-todos.md` (renamed) | NEW / RENAMED | Standalone docs. |
| `IMPLEMENTATION_PLAN.md`, `RESEARCH_MIGRATION_REFIT_KALSHI_ODDS.md`, `UNIFIED_RATE_ACCESS_TODOLIST.md` | NEW | Repo-root planning docs (+79, +839, +386). |
| `scripts/_archive/` | NEW dir, 75+ renamed/added files | The PR's archive consolidation — `scripts/archive` → `scripts/_archive` with subfolders `legacy_archive_orig/`, `odds_qa_testing/`, `debug/`, etc. Also pulls in three new files: `pipeline_walkthrough.py`, `qa_calibration_diff.py`, `qa_joint_fit_demo.py`. |
| `scripts/backtests/`, `scripts/eval/`, `scripts/debug/cluster/`, `scripts/odds_api_mapping/`, `scripts/odds_qa_testing/`, `scripts/migrations/_audit.py`, `scripts/scratch/` | NEW dirs | Live (non-archive) script trees: backtest harness CLI, eval harness, cluster-model debug, odds-API mapping, QA dashboards, migration-audit helper, scratch. |
| `.github/workflows/claude-code-review.yml` | DELETED (−78) | Removed CI workflow. |

### 5f. CLAUDE.md (+30 / −0)

Adds the **Starter-Only Assumption** block, the cluster-model open-inconsistency call-out, and the **"Redis is clustered in stage/prod — no native MGET across slots"** rule documenting the live incident on `get_market_odds_batch` (cited in `CLAUDE.md`).

---

## 6. Notable architectural decisions

1. **Six-folder service taxonomy.** `application/service/` flat layout → `{odds, calibration, backtest, betting, lineup, info}/`. 21 services, history preserved via `git mv`. The split is by *responsibility*, not by entry-point — calibration and odds are siblings, not one inside the other.
2. **Two alembic histories.** `alembic_ext_odds/` is now first-class (was `.ini` only before; the actual versions/ tree lands here). Migrations in BOTH must be run — explicitly documented in `.claude/rules/migrations.md`. **`alembic_soccer_data.ini` is deleted** in this PR — confirm no environment still depends on it before merging.
3. **New separate database schema `ext_odds`.** 23 tables in a dedicated PG schema, with its own alembic version table (`version_table_schema=SCHEMA_NAME` in `alembic_ext_odds/env.py:54, 79`). Populated by the sister project at `src/ext_odds_streaming/models/ext_odds_models.py`. This is a **new persistence subsystem**, not just a refactor.
4. **Source-aware data plane.** `events`, `match_statistics`, `player_statistics`, `lineups` all gain `source` columns (`whoscored` | `sofascore`). PKs/UNIQUEs widen to include `source`. This is a fundamental shape change to the read path: every consumer that did `scalar_one_or_none()` on `(game_id, player_id)` is at risk — `lineup_source_priority.py` exists specifically because some now hit `MultipleResultsFound`. The `core/data/event_source.py` constants module makes "which source is canonical for this game?" a first-class concept rather than a string literal. **Dependency direction is bottom-up**: helpers in `core/data/` are now imported by the `application/service/info/migration_service.py` and the migrators in `scripts/migrations/`.
5. **Calibration as a peer subsystem.** `MatchCalibrationService` (~2K LOC) elevates the previously ad-hoc factor adjustments into a layered service with its own enable flag, cooldowns, divergence threshold, extreme-policy alerting (`extreme_factor_alerts.py`), and Redis-backed adjustments cache. Three policies are configurable via `MATCH_CALIBRATION_EXTREME_POLICY`: clamp / passthrough / passthrough_with_alert.
6. **Vig math centralised.** `vig_policy.py` is the new home for divisive/power vig + inversion. Pure-functional, no I/O. Changes the dependency direction: pricing paths now depend on `application/service/odds/vig_policy.py`, eliminating duplicated overround code.
7. **External-API service layer.** `MarketOddsApiService` is the first attempt at collapsing the duplicated `fetch_*.py` scripts into a deep service with explicit error taxonomy (`OddsApiAuthError`, `OddsApiRateLimitError`, `OddsApiResponseError`). Layered DTOs (`Event`, `MarketSnapshot`, `PlayerPropSnapshot`) mark the boundary; persistence stays inside the service, not the caller.
8. **Backtest harness consolidation.** `BacktestRunner` + `BacktestScenario` enum + `_assert_no_leakage()` collapse three duplicated temporal-cutoff loops into one with leakage detection as a first-class assertion.
9. **W1 telemetry pipeline.** `RequestAnalyticsService` introduces a Kafka-backed background-thread consumer in the `application/service/info/` layer. This is the first time a service in `application/` runs an in-process consumer thread (existing Kafka consumers live in `infrastructure/kafka/`); worth flagging as a layering departure.
10. **Migration audit + idempotency gate.** `migration_runs` + `MigrationService.migrate_league()` add a 24h re-run gate (`--force` to override) for the SF↔WS migrators, with row-touched tracking in JSONB. The `r3s4t5u6v7w8` migration's destructive one-shot DELETE relies on this gate to be safe to roll forward repeatedly.
11. **Three open Alembic head merges in this PR** (`62a181c29a0b`, `6392e7d01782`) indicate the work was developed across multiple parallel branches (W2/W3/W4/W5 + audit-fix). Final head is `6392e7d01782`. Operators should expect to run `alembic upgrade head` for `alembic.ini` *and* `alembic -c alembic_ext_odds.ini upgrade head` for the new env.
12. **Strict autogenerate (`compare_type=True, compare_server_default=True`)** turned on for both alembic envs. Future `alembic revision --autogenerate` will surface column-type and server-default drift the codebase has historically tolerated. Expect noisier autogenerate output post-merge — review carefully.
13. **CI footprint shrunk.** `.github/workflows/claude-code-review.yml` removed (−78). Whatever automated review existed there is no longer running.
