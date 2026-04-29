# Arch Specialist — Synthesis (Mid Level)

> Synthesized from 4 leaf reports under `pr-668-leaves/`:
> [`arch-services.md`](pr-668-leaves/arch-services.md),
> [`arch-repositories.md`](pr-668-leaves/arch-repositories.md),
> [`arch-migrations.md`](pr-668-leaves/arch-migrations.md),
> [`arch-dtos-config.md`](pr-668-leaves/arch-dtos-config.md).

## TL;DR

Three new top-level namespaces (`services/sofascore`, `services/admin`, `domain/identity`, `integrations/`, `infrastructure/kafka/`); two heavily-modified existing services (`GameInfoService`, `UserBetService`); 22 alembic migrations (16 net-new tables + 6 merge heads + one **destructive** dedup); 8 new env vars (5 in `.required-env`, 8 in `.env.example` — drift on the ODDS_SERVICE_* set). The architecture follows the project's DDD layering with one notable exception: admin-side workflows use module-level mutable state plus daemon threads, not the async UoW pattern.

## 1. Services

### 1.1 New namespaces

| Namespace | Purpose | Files |
|---|---|---|
| `services/admin/` | Admin in-process job tracker + league-onboard service | 3 files |
| `services/sofascore/` | Read API for `/info/sofascore/*` (32 routes) + ID resolver | 3 files |

`SofascoreDataService` exposes ~22 read methods, all delegate to `SofascoreDataRepository`. `SofascoreIdResolver` handles the dual-input pattern (every route accepts UUID OR sf_id transparently).

### 1.2 Modified services

| File | Headline change |
|---|---|
| `services/statistics/game_info_service.py` | New `_compute_live_status_map` with partition + batched HTTP + time-based fallback |
| `services/bets/user_bet_service.py` | New `_emit_bet_placed_event` (Kafka topic `odds.user_bets`) |
| `services/notifications/event_processing_service.py` | Removed `_process_withdrawal_completed` + WITHDRAWAL_COMPLETED elif branch |
| `services/transactions/transaction_service.py` | Bug-3 fix — `create_manual_withdrawal` is now an instance method (was `@staticmethod` with `self`) |
| `services/withdrawals/withdrawal_service.py` | Modified — payment-metrics path retained |
| `services/notifications/{game_selection,process_outbox,queue_outbox}_service.py` | Modified |
| `services/{referrals,social}/...` | Modified |

Detail: see [`arch-services.md`](pr-668-leaves/arch-services.md).

### 1.3 Architectural conformance

- ✅ Application services depend on domain + repo ports.
- ✅ Identity abstractions live in `domain/identity/` with no infrastructure deps.
- ✅ Kafka schemas are `@dataclass` (not Pydantic) — deliberately decoupled from web layer.
- ⚠️ **`@flow_traced(Flow.X)` decoration on new services not verified** — should be confirmed as part of merge.
- ⚠️ `services/admin/league_onboard_service.py` uses module-level mutable state + sync DB inside daemon threads — diverges from the async UoW pattern used elsewhere (see risk leaf).

## 2. Repositories + UoW + ORM

### 2.1 New repository

`infrastructure/repositories/sofascore/sofascore_data_repository.py` — single read-side repo for all SofaScore routes. Async SQLAlchemy session, ORM-based queries against `sofascore_models.py`.

### 2.2 Modified repositories

`bets/{betslipbets,leg}_repository.py`, `promotions/promotion_repository.py`, `social/league_repository.py`, `statistics/team_info_repo.py`. Small changes; details not enumerated in this leaf.

### 2.3 ORM-first compliance

`.claude/rules/orm-first.md` listed model gaps for the SF + WS work in PRs #654 and #656. **All gaps are filled in this PR.** Verified ORM classes for:

```
sofascore_models.py:
  • SfTeamSquad           (line 987)
  • SfTeamPerformance     (line 1014)
  • SfMatchOddsFull       (line 1075)
  • ScrapeRun             (line 1113)
  • EnrichmentRun         (line 1158)
  • SfEnrichmentMatchState (line 1222)

whoscored_models.py:
  • WhPlayerResolutionAudit
  • WhoscoredRun
```

Detail: see [`arch-repositories.md`](pr-668-leaves/arch-repositories.md).

## 3. Migrations

### 3.1 Inventory

22 new alembic files. Composition:

| Type | Count | Notes |
|---|---|---|
| DDL (table create + column add) | 13 | Largest is `20260324_add_sofascore_tables_and_columns` |
| DDL + DML (destructive) | 1 | `20260420_dedup_competitions_add_unique` |
| DML (data fix) | 1 | `20260417_fix_england_national_league_sf_tid` |
| Schema-extend (FK / column / config) | 1 | `20260427_add_missing_sf_fks`, `20260428_add_config_to_audit_runs` |
| Merge heads (no-op) | 6 | Reflects parallel feature branches |

### 3.2 Headline migration: `20260420_dedup_competitions_add_unique`

```
upgrade():
   # Step 0: create competitions_dedup_audit (forensic table) — IF NOT EXISTS
   # Step 1: identify dupe groups via _comp_canonical TEMP TABLE
   #         canonical pick: (sf_unique_tournament_id IS NOT NULL) DESC, last_updated DESC NULLS LAST, competition_id ASC
   # Step 2: insert "plan_dedup_orphan" rows into audit BEFORE mutating
   # Step 3: re-point tournament_calendar.competition_id to canonical
   # Step 4: merge duplicate tournament_calendar rows (keep one per (canonical, sf_season_id, wh_season_id))
   # Step 5: cascade-delete orphan competitions
   # Step 6: ADD UNIQUE constraints on known_name + (wh_tournament_id WHERE NOT NULL)

downgrade():
   # drops UNIQUE constraints + drops audit table
   # does NOT restore deleted rows — recovery requires DB restore
```

### 3.3 Ordering invariant

`20260427_add_missing_sf_fks` MUST run AFTER `20260420_dedup_competitions` (otherwise the new FKs would violate against duplicate `competitions` rows). Enforced via the merge-heads chain.

### 3.4 ORM coverage

All 16 net-new tables have ORM classes in this PR. ✅ See §2.3.

### 3.5 Stale-column rule compliance

No migration in this PR drops a non-`_deprecated` column. ✅

Detail: see [`arch-migrations.md`](pr-668-leaves/arch-migrations.md).

## 4. DTOs / config

### 4.1 Schemas

| Layer | Items |
|---|---|
| Request (Pydantic, inline in routes) | `SfEnrichRequest`, `SfOnboardRequest`, `OnboardAction`, `TriggerJobRequest`, `TriggerJobResponse`, `JobResponse`, `JobStatus`, `MAX_RESULT_LENGTH` |
| Request (Pydantic, file) | `LeagueOnboardRequest`, `OnboardStatus`, `LEAGUE_PRESETS` in `application/schemas/requests/league_onboard_schemas.py` |
| Response (Pydantic, file) | `application/schemas/responses/sofascore/{__init__, info, league, match, player, team}.py` — **NOT wired** via `response_model=` |
| Microservice DTOs | `IsLiveBatchRequest`, `LiveStatusBatchResponse` |
| Kafka events (`@dataclass`) | `BetPlacedEvent`, `OddsRequestEvent` |
| Cross-service contracts (`TypedDict`) | `TargetedMigrationRequest`, `MigrationJobResponse` |

### 4.2 Config object additions

`config/core/app.py`:
- `LiveCheckMode` enum (ALWAYS, WINDOW).
- `LIVE_CHECK_MODE`, `LIVE_CHECK_WINDOW_BEFORE`, `LIVE_CHECK_WINDOW_AFTER`.
- `SECURE_COOKIES = (ENV == "prod")`.

### 4.3 Env-var changes

| Var(s) | In `.env.example`? | In `.required-env`? |
|---|---|---|
| `LIVE_CHECK_*` (3) | ✅ | ✅ |
| `KAFKA_*` (5) | ❌ (defaults in code) | ✅ |
| `ODDS_SERVICE_SYNC_*` (8) | ✅ | **❌ — drift** |

The 8 `ODDS_SERVICE_*` vars in `.env.example` are missing from `.required-env`, so SSM validation will not catch drift at deploy time. Risk specialist flag.

### 4.4 Integrations package (NEW)

`src/backend_server/integrations/` — distinct from `infrastructure/microservices/`. Convention emerging:

- `infrastructure/microservices/` — established RPC client (existing odds RPC, request/response DTOs).
- `integrations/` — cross-service "back-channel" clients (BS pushing data into BO via admin endpoints; hand-mirrored TypedDict contracts).

### 4.5 Domain identity package (NEW)

`src/backend_server/domain/identity/` — 7 files providing cross-provider ID abstractions (WhoScored ↔ SofaScore ↔ internal UUIDs). Critical for the SF coverage feature; consumed by both `services/sofascore/sofascore_id_resolver.py` and the scraper sinks.

### 4.6 Logging-formatter relocation (drift)

- New location: `src/backend_server/core/utils/logging_formatter.py`.
- `src/backend_server/app.py:54` — updated.
- `src/scheduler/main.py:34` — **NOT updated**, still imports from old path. Will fail at scheduler startup unless a backwards-compat re-export exists at the old path. Risk specialist flag.

Detail: see [`arch-dtos-config.md`](pr-668-leaves/arch-dtos-config.md).

## 5. Cross-cutting architectural notes

| Note | Detail |
|---|---|
| **Admin job pattern** | All three admin background workflows (`/admin/sofascore/enrich`, `/admin/sofascore/onboard`, `/admin/leagues/onboard`) use `threading.Thread(daemon=True)` + module-level `_jobs` dict. Status is lost on uvicorn worker restart. Inconsistent with the request-path async UoW pattern. |
| **Kafka decoupling** | `@dataclass` events plus a singleton producer keep Kafka concerns out of the web/domain layers. Producer is fire-and-forget by design — `produce` never raises. |
| **Routes-without-response-models** | 32 SofaScore data routes lack `response_model=`. Schemas exist; just not wired. Not a regression but a missed contract guarantee. |
| **Hand-mirrored cross-service DTO** | `_odds_admin_contracts.py` is a calculated trade-off — a TypedDict per BS↔BO admin endpoint, with an inline note proposing promotion to `specs/` if a second consumer appears. |
