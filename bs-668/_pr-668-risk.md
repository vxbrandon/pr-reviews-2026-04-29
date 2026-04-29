# Risk Specialist — Synthesis (Mid Level, Flow-Correctness Focus)

> Synthesized from 4 leaves under `pr-668-leaves/`:
> [`risk-migrations.md`](pr-668-leaves/risk-migrations.md), [`risk-observability-env.md`](pr-668-leaves/risk-observability-env.md), [`risk-flow-correctness.md`](pr-668-leaves/risk-flow-correctness.md), [`risk-cross-repo.md`](pr-668-leaves/risk-cross-repo.md).
>
> Per user direction, the previous `risk-auth-payment-concurrency` leaf was dropped. Auth posture for the BS→BO admin call is correct (JWT minted via `_mint_admin_jwt` with `role=admin`, 5-min lifetime). Payment-specific concerns (WITHDRAWAL_COMPLETED outbox in-flight) are folded into `risk-flow-correctness.md` §E as a flow concern.

## TL;DR

**4 hard blockers, 7 soft blockers** — the same count as before but the priorities have shifted. The headline flow-correctness risks are: (1) `request_id` / `X-Flow` NOT propagated on the BS→BO admin call; (2) 5 admin routes missing from `ROUTE_TO_FLOW`; (3) ContextVar loss in admin daemon threads; (4) logging_formatter import path drift in scheduler. None of these are correctness-fatal — they break observability, not data flow — but they all undermine the structured-logging story the rest of the codebase relies on.

## 1. Hard blockers (fix before merge)

### 1.1 `/admin/leagues/onboard*` × 5 routes missing from `ROUTE_TO_FLOW` (HIGH — observability rule)

`flow_context.ROUTE_TO_FLOW` (`infrastructure/middleware/flow_context.py`) lacks entries for:

```
POST   /admin/leagues/onboard
GET    /admin/leagues/onboard
GET    /admin/leagues/onboard/{job_id}
DELETE /admin/leagues/onboard/{job_id}
POST   /admin/leagues/onboard/preset/{preset_name}
```

Effect: `flow_ctx` is None for these requests; logs lack `flow` field; cross-service `X-Flow` header is omitted on any downstream call. Pre-commit drift hook `./scripts/check-flow-drift.sh` exits 1.

Fix: 5 lines.

Source: [`risk-observability-env.md`](pr-668-leaves/risk-observability-env.md) §A, [`risk-flow-correctness.md`](pr-668-leaves/risk-flow-correctness.md) §A.1.

### 1.2 `request_id` / `X-Flow` not propagated to BO on admin call (HIGH — flow correctness)

```python
# integrations/odds_service_client.py:103-109
def _admin_auth_headers() -> dict[str, str]:
    token = _mint_admin_jwt()
    return {"Authorization": f"Bearer {token}"} if token else {}
```

This is the only header dict sent to `BO /odds/admin/migration/run-targeted`. Compare with `OddsClient._propagation_headers()` which forwards all 7 ContextVar-derived headers via `get_propagation_headers()`.

Effect: BS scheduler tick with `request_id_ctx=abc123, flow_ctx=data.live_fetch` triggers a POST to BO that lands in BO's logs without the `X-Request-ID` header. Cross-service trace breaks at the scheduler→BO boundary. The `OddsRequestEvent` Kafka emit IS correctly correlated, but the targeted-migration HTTP call is not.

Fix:
```python
def _admin_auth_headers() -> dict[str, str]:
    headers = get_propagation_headers()
    token = _mint_admin_jwt()
    if token:
        headers["Authorization"] = f"Bearer {token}"
    return headers
```

Source: [`risk-flow-correctness.md`](pr-668-leaves/risk-flow-correctness.md) §A.4.

### 1.3 `src/scheduler/main.py:34` imports old `logging_formatter` path (HIGH — scheduler may break)

```python
# scheduler/main.py:34 (UNCHANGED)
from backend_server.logging_formatter import configure_logging   ← OLD path

# app.py:54 (UPDATED) and middleware/request_id.py:7 (UPDATED)
from backend_server.core.utils.logging_formatter import configure_logging   ← NEW path
```

Per `.claude/rules/deploy.md` `main + scheduler` deploy together — a broken scheduler blocks the deploy. Either update the import or add a backwards-compat shim at the old path.

Source: [`risk-observability-env.md`](pr-668-leaves/risk-observability-env.md) §E.

### 1.4 `20260420_dedup_competitions_add_unique` not smoke-tested on stage (HIGH — destructive)

Deletes orphan rows, re-points `tournament_calendar` FKs, merges duplicate calendar rows, then adds UNIQUE constraints. Audit-table mitigates traceability, but downgrade does NOT restore deleted rows — recovery requires DB restore.

Migration docstring also notes a follow-up `base_sink._resolve_legacy_id` PR is required to keep the JSON mapping in sync. **The follow-up is NOT in this PR**. Mid-flight scrape after dedup migration but before `base_sink` follow-up could fail to resolve `competition_id` for some leagues.

Source: [`risk-migrations.md`](pr-668-leaves/risk-migrations.md) §B, [`risk-flow-correctness.md`](pr-668-leaves/risk-flow-correctness.md) §D.1.

### 1.5 PR description checklist all blank (HARD — rule violation)

```
[ ] Unit tests added/updated
[ ] Integration tests added/updated
[ ] Manual QA performed
[ ] Edge cases considered
[ ] ⚠️ IMPORTANT: Backtest with previous dev launch ⚠️
```

The "Backtest with previous dev launch" item is explicitly required by the project template.

Source: [`risk-observability-env.md`](pr-668-leaves/risk-observability-env.md) §H.

## 2. Soft blockers (strongly recommended)

### 2.1 ContextVar loss in admin daemon threads (MED — flow correctness)

Three admin endpoints spawn `threading.Thread(daemon=True)` without `contextvars.copy_context()`:

```
POST /admin/sofascore/enrich         → _run_sf_enrich_sync
POST /admin/sofascore/onboard        → _run_sf_onboard_sync
POST /admin/leagues/onboard{,/preset/{name}}  → _run_onboard_workflow
```

Logs from these threads lack `request_id`, `flow`, `user_id`. Trace correlation breaks at the thread boundary.

Fix:
```python
import contextvars
ctx = contextvars.copy_context()
threading.Thread(target=lambda: ctx.run(_run_sf_*_sync, *args), daemon=True).start()
```

Source: [`risk-flow-correctness.md`](pr-668-leaves/risk-flow-correctness.md) §A.3.

### 2.2 New SF / admin services NOT decorated with `@flow_traced` (MED)

`grep '@flow_traced' application/services/sofascore/*.py application/services/admin/*.py` returns 0. Per CLAUDE.md "How to: Add a new API endpoint" step 4, every service entry point should be decorated. Other services (`bets/user_bet_service.py`, `chat/chat_service.py`) consistently use it.

Effect: middleware sets `flow_ctx` so log-line tagging works, but service-method-level OTEL spans are missing for the new routes.

Source: [`risk-flow-correctness.md`](pr-668-leaves/risk-flow-correctness.md) §A.2.

### 2.3 `ODDS_SERVICE_SYNC_*` env-var drift (MED)

8 vars in `.env.example`, **0** in `.required-env`. SSM validation will not catch a misconfigured `ODDS_SERVICE_URL` in prod (default points to `localhost:8101`).

Source: [`risk-observability-env.md`](pr-668-leaves/risk-observability-env.md) §F.

### 2.4 BO endpoint deploy ordering (MED — cross-repo)

| Required before | Symptom otherwise |
|---|---|
| BO `/live/is_live_batch` shipped | `is_live` checks silently fall back to time-based; live-betting tile correctness regresses |
| BO `/odds/admin/migration/run-targeted` shipped | Targeted-migration POST 404s (logged warning); BO stays stale on lineups |
| BO Kafka consumers ready | Events queue in topic until consumed |

Source: [`risk-cross-repo.md`](pr-668-leaves/risk-cross-repo.md) §C.

### 2.5 `WITHDRAWAL_COMPLETED` outbox in-flight rows (MED)

`notif_outbox` rows with `event_type='withdrawal_completed'` and `status<>'sent'` will fall through the `else` branch logging "Unknown event type". Pre-deploy SQL check + decide policy.

Source: [`risk-flow-correctness.md`](pr-668-leaves/risk-flow-correctness.md) §E.

### 2.6 Hand-mirrored TypedDict drift (MED — cross-repo)

`integrations/_odds_admin_contracts.py` is a hand-mirrored copy of BO's admin endpoint shapes. No automated drift detection. The day this PR merges, manually diff against BO's actual handler signatures.

Source: [`risk-cross-repo.md`](pr-668-leaves/risk-cross-repo.md) §A.1.

### 2.7 BS-side cross-tick deduplication relies on BO (MED)

`_fetch_recently_scraped_wh_game_ids(window=15min)` re-queries the same window on every tick. At 90s `prelive_lineups_imminent` cadence, ~90% of `game_ids` overlap with the previous tick. BS does not dedupe — relies on BO `/odds/admin/migration/run-targeted` being idempotent on the `game_ids` list. Cross-repo question for the BO team.

Source: [`risk-flow-correctness.md`](pr-668-leaves/risk-flow-correctness.md) §C.4.

## 3. Operational notes (track post-merge)

| Note | Detail |
|---|---|
| Multi-instance scheduler scale-out unsafe for `SingleInstanceLock` | Lock is process-local. Two scheduler ECS tasks would both fire scrape jobs concurrently. Sinks tolerate via ON CONFLICT, BO tolerance unverified. |
| Background-thread admin job persistence | Module-level `_jobs` dict is lost on uvicorn worker restart and is per-worker. Cross-worker GET .../jobs/{job_id} may 404. Follow-up: Redis/DB. |
| `prelive_lineups_imminent` 90s cadence | ~960 BS→BO requests/day. Verify BO load capacity. Add Grafana panel for `enrichment_runs.status='failed' WHERE tier='live'` (Slack disabled for this tier). |
| `OddsClient.DEFAULT_TIMEOUT 30s→5s` | Latency spikes on BO surface more `OddsMicroserviceRequestError` on parlay/market endpoints (no time-based fallback for those). |
| Public `/info/sofascore/*` × 32 routes | Unauthenticated; no rate-limit. SofaScore upstream quota and DB read capacity exposure. Follow-up: rate-limit if abuse appears. |
| 89k-line PR is hard to revert atomically | Stage-rollout order: SF data routes → admin SF routes → BO sync flag → Kafka producer flag. |
| Multiple alembic merge-heads | Verify `alembic heads` returns ONE head from clean DB before merge. |

## 4. WebSocket protocol stability — non-issue

PR #668 introduces zero WebSocket flows. Cross-repo WS contract risk vs Backend-Odds is non-existent for this PR.

Source: [`risk-flow-correctness.md`](pr-668-leaves/risk-flow-correctness.md) §B.

## 5. Migration safety as it affects flows

| Migration | Affected flow(s) | Severity |
|---|---|---|
| `20260420_dedup_competitions_add_unique` | `/info/get_games_in_date_range`, `/info/sofascore/league/*`, scraper `BaseSink._resolve_legacy_id` | HIGH (depends on follow-up `base_sink` PR) |
| `20260427_add_missing_sf_fks` | scrape pipelines (FK violations) | LOW once dedup completes |
| `20260324_add_sofascore_tables_and_columns` | `/info/sofascore/*` | LOW (new tables; null reads handled) |
| Other DDL (10+) | none directly | LOW |

Source: [`risk-flow-correctness.md`](pr-668-leaves/risk-flow-correctness.md) §F.

## 6. TODO/FIXME debt added

| File | Note |
|---|---|
| `scraper_job_registry.py` | `TODO: when run_targeted_match_pipeline gains minutes-granularity filtering, tighten this to a 30-min window.` (`prelive_lineups_imminent`) |
| `scraper_job_registry.py` | Two `FIXME(task-5/6)` comments noting pre-existing broken `dq_audit` / `bet_audit` imports. Explicitly NOT fixed in this PR. |

## 7. Pre-existing risk surfaced (not introduced)

- `.env.example` contains real-looking Stream API secret + Slack tokens (committed in `643f376e`). Out of scope for this PR but worth rotating.

## 8. Action plan (priority order)

1. **Add 5 entries to `ROUTE_TO_FLOW`** for `/admin/leagues/onboard*`. *(HARD)*
2. **Add propagation headers to BS→BO admin call** — modify `_admin_auth_headers()` to merge `get_propagation_headers()`. *(HARD — flow correctness)*
3. **Update `src/scheduler/main.py:34`** to new `logging_formatter` path, OR add a re-export shim. *(HARD)*
4. **Smoke-test `20260420_dedup_competitions_add_unique`** on stage-snapshot DB (upgrade → check audit table → downgrade → upgrade), AND coordinate with the follow-up `base_sink` PR. *(HARD)*
5. **Fill the PR description checklist** with green-test paste + backtest evidence. *(HARD)*
6. **Wrap admin daemon threads with `contextvars.copy_context()`**. *(SOFT)*
7. **Add `@flow_traced` to new SF + admin service methods**. *(SOFT)*
8. **Add `ODDS_SERVICE_SYNC_*` to `.required-env`** with `[optional]` markers. *(SOFT)*
9. **Confirm BO endpoint readiness** via cross-team comms before each flag flip. *(SOFT)*
10. **Verify `Flow.DATA_SOFASCORE` and `Flow.ADMIN_SCRAPER`** exist in `core/observability/flows.py` AND in `specs/flow-registry.json`. *(SOFT)*
11. **Pre-deploy SQL check** on in-flight `notif_outbox.event_type='withdrawal_completed'` rows. *(SOFT)*
12. **Manual contract diff** of `_odds_admin_contracts.py` vs BO's handler signatures on merge day. *(SOFT)*
13. **Verify `alembic heads` returns one head** post-rebase. *(SOFT)*
14. **Verify BO is idempotent** on the `game_ids` list at `/odds/admin/migration/run-targeted`. *(SOFT — cross-repo question)*
