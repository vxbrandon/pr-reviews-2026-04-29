---
pr: 140
repo: TBG-AI/Backend-Odds
branch: dev-coverage-and-odds-impro
base: dev
reviewer: risk-and-blocker-specialist
date: 2026-04-29
focus: flow-correctness risk (migration safety, observability, cache, cross-service flow contracts)
---

# PR #140 — Flow-Risk Audit

Scope (per team-lead direction): risks that break **flows** between services
or inside a flow's runtime. Migration safety to flows, observability gaps in
flow paths, Redis/CROSSSLOT issues that affect flow runtime, and cross-repo
flow contracts (Kafka schemas, WebSocket protocol, request_id/X-Flow header
propagation, shared Redis keys). Auth/cookie/JWT items moved to footnote.
Read-only audit.

330 files changed; 232 source/test/migration files reviewed by grep+spot-read.
Built from `git diff <merge-base>..HEAD` against `origin/dev` (merge base
`1b5fb0aa6afca056e80b78622a85f4dcf9940687`); `gh pr diff` 406s on this PR.

---

## 1. Blockers (must fix before merge)

### B1. Default-on alembic upgrade in entrypoint will crash-loop the *flow runtime* on rolling deploy
- File: `scripts/entrypoint.sh:25, 43-47`
- Before: `RUN_MIGRATIONS` was not handled by entrypoint.
- After: `RUN_MIGRATIONS="${RUN_MIGRATIONS:-true}"` runs `alembic upgrade head`
  for *both* `alembic.ini` and `alembic_ext_odds.ini` on every container start
  before `exec "$@"`.
- Why this is a flow risk:
    - The new `q2r3s4t5u6v7_add_event_source_tracking.py` migration
      (`alembic/versions/q2r3s4t5u6v7_add_event_source_tracking.py:110-133`) executes
      *row-level dupe pre-checks* and `raise RuntimeError(...)` if it finds
      duplicates on `match_statistics(game_id, team_id, source)` or
      `player_statistics(player_id, game_id, source)`. If those checks fail,
      every replacement task crashes before it reaches uvicorn.
    - The same migration drops + re-adds the PRIMARY KEY on
      `match_statistics` and the UNIQUE on `player_statistics` via
      `ALTER TABLE … ADD CONSTRAINT … USING INDEX` (lines 154-174) — these
      tables are read by *every* odds-pricing flow (ZAP/ZAT/ZAO).
    - On ECS rolling restart, all replacement tasks try the upgrade in
      parallel; alembic's advisory lock serializes the writes, but a raise in
      `q2r3s4t5u6v7` propagates to every container = simultaneous flow
      brownout.
- Suggested action: default `RUN_MIGRATIONS=false` and run alembic from a
  dedicated one-shot ECS task / `docker-compose.migrate.yml` *before* the
  service rollout. The repo rule `.claude/rules/migrations.md` already
  requires this pattern (`./scripts/run_with_env.sh local "alembic upgrade
  head"`) — entrypoint migrations contradict it.

### B2. Env-var drift breaks the deploy gate that protects flow startup
- Rule: `.claude/rules/environment.md` — every `os.getenv` / `os.environ.get`
  / `os.environ[...]` in `src/` must have a matching entry in `.required-env`,
  validated by `deployment/scripts/validate-env.sh` (warns dev/stage, blocks
  prod on missing `[required]`).
- 22 vars used in this PR (in `src/` or `scripts/`) are missing from
  `.required-env`. Vars **inside `src/`** (deploy-gate-relevant):

| Var | First-seen file:line context |
|-----|------------------------------|
| `ALEMBIC_DRIFT_FAIL` | `src/backend_odds/infrastructure/migrations/drift_check.py`, `src/backend_odds/infrastructure/web/app.py` |
| `CALIBRATION_ENABLE_CALIBRATE` | `src/backend_odds/infrastructure/api/tasks.py` |
| `CALIBRATION_FORCE_REFIT` | `src/backend_odds/infrastructure/api/tasks.py` |
| `MATCH_CALIBRATION_FACTOR_MIN` / `_MAX` / `_WARN_LO` / `_WARN_HI` | `src/backend_odds/application/service/calibration/match_calibration_service.py` |
| `MATCH_CALIBRATION_PLAYER_PROP_FACTOR_MIN` / `_MAX` / `_WARN_LO` / `_WARN_HI` | `src/backend_odds/core/utils/runtime_flags.py`, `match_calibration_service.py` |
| `MATCH_CALIBRATION_TEAM_PROP_FACTOR_MIN` / `_MAX` / `_WARN_LO` / `_WARN_HI` | `match_calibration_service.py` |
| `MATCH_CALIBRATION_TEAM_PROP_GAMMA` / `_HOLD` / `_MAX_AGE_HOURS` | `match_calibration_service.py` |
| `ODDS_FETCH_PLAYER_PROPS_MAX_EVENTS` | `src/backend_odds/infrastructure/kafka/scheduled_odds_fetch.py` |

- Why this is a flow risk: `MATCH_CALIBRATION_*` factor bounds gate the
  per-match calibration adjustment that rides the live-odds flow; if these
  drift silently between dev/stage/prod the calibration arm of the flow
  produces different numbers per environment with no deploy-time visibility.
- Suggested action: add each `src/`-referenced var to `.required-env` as
  `[optional]` with the matching default. Script-only vars
  (`PLAYER_PROPS_REDIS_TTL`, `URA_APP_PORT`, `VERBOSE`, `VERIFY_LOG_LEVEL`)
  are out of scope if those scripts never run in the prod container — confirm
  with deploy contract.

### B3. Hardcoded prod RDS credential in committed migration script
- File: `scripts/migrations/async_migrate_whoscored_new_to_postgres_v2.py:2329`
- `os.getenv("ASYNC_SOCCERDATA_DATABASE_URL", "postgresql+asyncpg://tbg:<REDACTED>@<prod-db-cluster-REDACTED>.us-east-1.rds.amazonaws.com:5432/new_db_prod")`
- Why this is a flow risk: the migration script is the data-flow source for
  WhoScored events into BO. A leaked credential lets anyone with repo read
  access write to (or read from) the upstream prod DB — corrupting every
  downstream flow that depends on `events` / `match_statistics`. Even if the
  password is stale, the hostname + username inventory is published.
- Suggested action: delete the fallback default, require
  `ASYNC_SOCCERDATA_DATABASE_URL` to be set, and rotate the credential in
  SSM. Audit `async_migrate_whoscored_new_to_postgres.py:1748` (older copy)
  for the same string and rotate together.

---

## 2. Strong concerns (should fix; document if not)

### S1. New flow paths have no `@flow_traced` / `@traced` annotations
- The diff's `^\+.*@flow_traced` / `^\+.*@traced` is empty across `src/`.
- New service classes and scheduled flow drivers running without flow
  context:
    - `src/backend_odds/application/service/calibration/match_calibration_service.py:524`
      `class MatchCalibrationService` — drives the W2 per-match calibration
      arm of the live-odds flow.
    - `src/backend_odds/application/service/odds/market_odds_api_service.py:83`
      `class MarketOddsApiService` — fans market odds into the pricing
      pipeline.
    - `src/backend_odds/application/service/info/data_completeness_service.py:70`
      `class DataCompletenessService` — feeds `/admin/data-health/*`.
    - `src/backend_odds/application/service/info/migration_service.py:75`
      `class MigrationService` — drives `/admin/migrate/{league_code}`.
    - `src/backend_odds/application/service/backtest/backtest_runner.py:234`
      `class BacktestRunner`.
    - Scheduled tasks: `run_match_calibration_check` (`tasks.py:1133`),
      `run_lineup_scraper` (`tasks.py:813`), `run_scheduled_odds_fetch`
      (`scheduled_odds_fetch.py:256`), `_exposure_snapshot_loop`
      (`web/app.py:349`).
- Why this is a flow risk: `.claude/rules/observability.md` requires service
  entry points to be `@flow_traced(Flow.X)`-decorated so Loki/Tempo can
  correlate. Without it, when the calibration arm of the live-odds flow
  misbehaves (e.g. produces a 5x adjustment), Grafana can't tie the bad
  pricing event back to the calibration call — flow correlation is broken
  for every new path.
- Suggested action: add `@flow_traced(Flow.ODDS_COMPUTE)` (or the closest
  flow enum) to public methods on each service class. For scheduled tasks
  use a manual span (`tracer.start_as_current_span("scheduled.X")`) per
  CLAUDE.md "How to: Add WebSocket/Kafka spans". Verify the `Flow.X` enum
  exists in `core/observability/flows.py` and `specs/flows/flow-registry.json`
  (run `./scripts/check-flow-drift.sh`).

### S2. New routes missing from `ROUTE_TO_FLOW` — flow tag absent on every request to them
- File: `src/backend_odds/infrastructure/web/middleware/request_context.py:42-130`
- Only 2 new routes added:
    - `("GET", "/admin/data-health/games/{game_id}"): "admin.operations"`
    - `("GET", "/admin/data-health/incomplete"): "admin.operations"`
- New routes added in this PR that are **not** in `ROUTE_TO_FLOW`:
    - `POST /admin/migration/run-targeted`, `GET /admin/migration/{job_id}`
    - `GET /admin/config`, `POST /admin/config/{name}`,
      `DELETE /admin/config/{name}`, `GET /admin/config/ui`
    - `POST /admin/pipelines/{name}/assign-and-fit`,
      `POST /admin/pipelines/{name}/post-process`
    - `POST /admin/reload-season-map`, `GET /admin/sources/actions`
    - `POST /admin/migrate/{league_code}`
- Why this is a flow risk: `RequestContextMiddleware` reads
  `ROUTE_TO_FLOW` to set the `flow` ContextVar that `TbgJsonFormatter` writes
  into every JSON log line and `httpx` propagates downstream as `X-Flow`. A
  missing entry → `flow=` is empty → any cross-service call from these
  endpoints to Backend-Server arrives without a flow tag, breaking
  per-flow Loki / Tempo correlation across the service boundary (cross-repo
  flow contract).
- Suggested action: add an entry per route. `admin.operations` is fine for
  migration/config/pipelines; if migrations should land under a more
  specific flow (`admin.migration`), match what BS uses.

### S3. Redis keys in flow runtime — verify cluster-safety on the lineup-detector path
- File: `src/backend_odds/application/service/lineup/lineup_change_detector.py`
  (consumed via `infrastructure/web/app.py:325`).
- The lineup-scraper flow (W4) writes lineup-state keys per game and reads
  them back to detect changes. The repo rule (CLAUDE.md "Redis is clustered
  in stage/prod") forbids native `MGET` / `_client.pipeline()` on multi-slot
  keys. The new code uses
  `client.zadd` / `client.zrevrange` / `client.zremrangebyrank` against a
  *single* key (`EXTREME_FACTOR_QUEUE_KEY` in
  `src/backend_odds/application/service/odds/extreme_factor_alerts.py:90,96,121`)
  — that's single-slot and safe.
- Verified safe: `RedisCacheRepository.mget` / `mset`
  (`src/backend_odds/infrastructure/cache/redis_repo.py:80-138`) loop per-key
  with comment "stage/prod run Redis in cluster mode … we can't use native
  MGET" — explicitly cluster-safe.
- Verified safe call site: `players_repository.py:86`
  (`cached_values = self.cache.mget(cache_keys)`) routes through the safe
  wrapper.
- Needs reviewer eyes:
    - `LineupChangeDetector` — the diff didn't show its key shape; if it
      ever accumulates a list of `lineup:{game_id}` keys and tries
      `_client.mget(keys)` directly, that path will CROSSSLOT in prod. Worth
      a focused read before merge.
    - Any new code in `infrastructure/api/websocket/manager_typed.py` /
      `manager.py` that reads multiple game-keyed cache entries.

### S4. Background flow tasks unconditionally `asyncio.create_task` even when the feature flag is off
- File: `src/backend_odds/infrastructure/web/app.py:292, 305, 329, 366`
- `odds_fetch_task` / `match_cal_task` / `lineup_scraper_task` / `exposure_task`
  are created at app startup regardless of their `*_ENABLED` flag.
- Mitigation in place: each task body early-returns when its flag is unset
  (`tasks.py:837` for `LINEUP_SCRAPER_ENABLED`, `tasks.py:1153` for
  `W2_ENABLED`, `scheduled_odds_fetch.py:277` for `ODDS_FETCH_ENABLED`), so
  no actual flow work runs. Cosmetic at runtime — but if someone adds
  pre-flag-check code later (e.g., setup logging, open a Redis connection),
  the gate breaks silently.
- Suggested action: skip the `create_task` call entirely when the flag is
  unset, so the gather/cancel list at `web/app.py:372-374` doesn't include
  no-op coroutines.

### S5. Both event-source-tracking migrations DELETE on downgrade — silent flow data loss
- Files:
    - `alembic/versions/q2r3s4t5u6v7_add_event_source_tracking.py:187-188`
      (`DELETE FROM match_statistics WHERE source = 'sofascore'`,
      `DELETE FROM player_statistics WHERE source = 'sofascore'`)
    - `alembic/versions/s4t5u6v7w8x9_add_source_to_lineups.py:106`
      (`DELETE FROM lineups WHERE source = 'sofascore'`)
- Why this is a flow risk: SF data is consumed by every fitting + pricing
  flow that reads `match_statistics` / `player_statistics` / `lineups`. If
  ops rolls back to recover from an unrelated bug *after* SF ingest has
  started, all SF rows vanish silently and downstream flows continue against
  a pre-SF baseline with no signal that data was deleted.
- Suggested action: gate the DELETE behind a `CONFIRM_DOWNGRADE_DATA_LOSS=1`
  env var, or write a runbook entry that rollbacks must be paired with an
  SF-migrator re-run.

### S6. Migration revision-id mismatch in docstring (cosmetic but indicates copy-paste)
- File: `alembic/versions/mig_audit_202604230358_add_migration_runs_audit_table.py:7`
- Docstring says `Revision ID: r3s4t5u6v7w8` but the assignment at line 15 is
  `revision = "mig_audit_202604230358"`. Alembic uses the assignment, so
  this is harmless — but worth a sanity-check on the body too. (Body
  inspected: creates `migration_runs` table + lookup index, downgrade drops
  both — looks correct.)
- Suggested action: fix the docstring to match.

---

## 3. Nice-to-haves

### N1. `compare_type=True, compare_server_default=True` enabled on both alembic envs
- Files: `alembic/env.py:60-83`, `alembic_ext_odds/env.py:53-81`. Good
  defensive change — catches type/default drift in autogenerate. No CI test
  asserts `alembic check` emits zero diffs against `target_metadata`;
  consider adding so model-vs-migration drift can't sneak in and silently
  break a flow's expected schema.

### N2. New `f0a1b2c3d4e5_initial_ext_odds_schema.py` is hand-written, 989 lines
- File: `alembic_ext_odds/versions/f0a1b2c3d4e5_initial_ext_odds_schema.py:1-15`
  (docstring: "manually constructed" baseline). Risk surface for `ext_odds.*`
  flows that depend on autogenerate-equivalent shape. Validate via
  `tests/unit/alembic/test_ext_odds_baseline_migration.py` (needs reviewer
  eyes — does it assert full shape?) or run an autogenerate dry-run on an
  empty staging DB and diff.

### N3. `mset` in `redis_repo.py` traces only at the wrapper
- File: `src/backend_odds/infrastructure/cache/redis_repo.py:116-138`. The
  per-key `set` calls are not child spans, so a slow individual key on the
  flow path won't surface in Tempo. Probably fine given cardinality, but
  consider attribute-only logging of (max key, max duration) if the flow
  ever hits a slow key.

### N4. `# noqa: SLF001` accesses `RedisCacheRepository._client` directly
- Examples: `extreme_factor_alerts.py:90, 119`, `web/app.py:325`. These are
  legitimately accessing the underlying client for ZSET ops not exposed by
  the wrapper. If the team wants to keep `_client` private, expose
  `get_raw_client()` or `zadd`/`zrange` wrappers — otherwise every caller
  has to repeat the noqa. Not a flow risk.

---

## 4. Cross-repo flow-contract risks

### X1. New Kafka topic `odds.lineup_updates` — schema contract with downstream consumers
- Files: `src/backend_odds/infrastructure/kafka/lineup_producer.py`,
  `src/backend_odds/infrastructure/kafka/schemas.py:LineupUpdate`
  (added in this PR — diff shows new `OddsUpdate` defensive deserialization
  with bounded dedup + counter at `schemas.py:24-50`).
- Default topic name (`KAFKA_LINEUP_TOPIC=odds.lineup_updates`) is published
  by the W4 lineup-scraper flow (`web/app.py:329`).
- Why this is a flow risk: any Backend-Server (or other) consumer reading
  this topic must accept the `LineupUpdate` dataclass shape (game_id,
  home_starters, home_bench, away_starters, away_bench, home_team_id,
  away_team_id, …). If BS subscribes to it (or plans to), the schema must
  be locked in `specs/` *before* this flag is flipped on in prod.
- Action: needs reviewer eyes — confirm with BS team whether
  `odds.lineup_updates` has a downstream consumer, and if so whether the
  schema matches. The defensive `unknown_field` counter on
  `OddsUpdate.from_dict` (`schemas.py:24-50`) is a good pattern; the same
  posture should be mirrored on the BS side.

### X2. Scheduled `odds_fetch` writes Redis keys consumed by the live-odds flow
- File: `src/backend_odds/infrastructure/kafka/scheduled_odds_fetch.py:256+`
- W3 task writes `market:odds:{game_id}` (and similar) keys consumed by the
  WS gateway and by Backend-Server's live-odds reader. Currently
  `ODDS_FETCH_ENABLED=false` by default, so no behavior change ships — but
  flipping it in `.env.prod` starts a 5-min polling loop that becomes the
  *upstream* of every downstream live-odds flow.
- Action: do not flip the flag without a runbook entry covering: (a) which
  Redis keys/TTLs the flow writes, (b) which BS consumers read them,
  (c) backfill / rollback steps if the polling loop emits bad data.

### X3. Cross-service `X-Request-ID` / `X-Flow` propagation depends on `ROUTE_TO_FLOW` completeness
- `RequestContextMiddleware`
  (`src/backend_odds/infrastructure/web/middleware/request_context.py`) reads
  `X-Request-ID` + `X-Flow` from incoming BS calls and sets ContextVars.
  Outbound httpx requests carry them back. Per S2 above, ~10 new admin
  routes are missing from `ROUTE_TO_FLOW` — for those endpoints, BO will
  still receive the inbound headers correctly, but any *outbound* call from
  those handlers will lose the `flow=` tag because it's never set in the
  ContextVar. End-to-end correlation breaks one hop in.
- Action: same as S2 — add the entries.

### X4. WebSocket protocol — no breaking changes detected
- File: `src/backend_odds/infrastructure/api/websocket/manager.py`,
  `manager_typed.py`, `handler.py`. Diff shows no new/changed
  `send_json` / `send_text` / message-type fields. Existing BS WebSocket
  clients should be unaffected.

### X5. Admin route auth contract change (footnote — deprioritized per team-lead)
- `src/backend_odds/infrastructure/api/rest/routes.py` now applies
  `Depends(verify_admin_token)` to `admin`, `pipeline_admin`, `data_health`
  routers. Any pre-existing BS → BO admin call without `Authorization:
  Bearer …` will now 401. Handled by ops as part of the rollout, not a
  flow-correctness risk.

---

## Summary

- **3 blockers**: entrypoint default-on alembic upgrade (B1), `.required-env`
  drift on flow-runtime config (B2), hardcoded prod creds in migration script (B3).
- **6 strong concerns**: missing `@flow_traced` (S1) and `ROUTE_TO_FLOW` (S2)
  break flow correlation; needs-eyes Redis cluster-safety check on
  `LineupChangeDetector` (S3); cosmetic background-task gating (S4);
  silent SF data loss on downgrade (S5); migration docstring revision-id
  typo (S6).
- **4 nice-to-haves**: alembic drift CI, hand-written ext_odds baseline
  validation, redis tracing depth, `_client` wrapper hygiene.
- **5 cross-repo flow-contract risks**: new `odds.lineup_updates` Kafka
  topic schema (X1), `odds_fetch` Redis keys upstream of live-odds flow
  (X2), `X-Flow` propagation depends on `ROUTE_TO_FLOW` completeness (X3),
  WebSocket protocol unchanged (X4), admin auth contract changed (X5,
  footnote).

**Verified safe**: alembic chain is single-headed (`6392e7d01782`);
CONCURRENT index work is correctly factored with `autocommit_block` +
`USING INDEX` constraint promotion; `RedisCacheRepository.mget` / `mset` are
cluster-safe loops; players_repository batch read routes through the safe
wrapper; starter-only assumption preserved in the new `BacktestRunner`
(default `starter_only=True`, `backtest_runner.py:111`),
`scripts/eval/run_eval.py:763`, and
`scripts/debug/cluster/evaluate_cluster_model.py:89`.

**Recommended next step**: address B1 (move alembic out of entrypoint) and
B3 (rotate + remove fallback creds) before merge. B2 should land in a
follow-up commit on this PR; once `.required-env` is correct, S2 and S1
become a small mechanical pass and unblock end-to-end flow correlation.
