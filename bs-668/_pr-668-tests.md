# Tests Specialist — Synthesis (Mid Level)

> Synthesized from 3 leaf reports under `pr-668-leaves/`:
> [`tests-unit.md`](pr-668-leaves/tests-unit.md),
> [`tests-integration.md`](pr-668-leaves/tests-integration.md),
> [`tests-coverage-gaps.md`](pr-668-leaves/tests-coverage-gaps.md).

## TL;DR

PR #668 adds **62 test files / ~10,705 LOC**: strong coverage of new SF data layer, named bug fixes, and `_compute_live_status_map`. Significant gaps in Kafka emit verification, cross-service `odds_service_client`, response-model enforcement, and migration safety. PR description checklist is **all blank** including the explicit "backtest with previous dev launch" box.

## 1. Coverage at a glance

| Bucket | Files | LOC | Marker |
|---|---|---|---|
| `tests/unit/sofascore/` | 14 | ~4,648 | unit |
| `tests/unit/whoscored/` | 4 | ~878 | unit |
| `tests/unit/{audit,admin,mappings,sinks}/` | 6 | small | unit |
| `tests/unit/test_critical_bug_fixes.py` | 1 | 211 | unit (AST-based) |
| `tests/unit/test_security_fixes.py` | 1 | 213 | unit (AST + behaviour) |
| `tests/info/unit/` (added files) | 4 | ~539 | unit |
| `tests/flow_tests/sofascore/` | 3 | 1,013 | `@pytest.mark.flow` |
| `tests/flow_tests/admin/test_sofascore_routes.py` | 1 | 209 | `@pytest.mark.flow` |
| `tests/flow_tests/run_cross_service_sf.sh` | 1 (shell) | n/a | manual / CI orchestration |
| `tests/qa_test/odds-pricing-main/` | 12+ shell + 2 Python | n/a | operational verification (not pytest) |
| `tests/{notification,payment,fixtures}/` | 3 (small) | small | mixed |

## 2. Test-to-code mapping (high-value rows)

| Code | Test |
|---|---|
| `OddsClient.get_live_status_batch` | `test_odds_client_live_batch.py` |
| `GameInfoService._compute_live_status_map` | `test_live_status_batch.py`, `test_game_info_service_live_status.py` |
| `auth_routes` `SECURE_COOKIES` cookie hardening | `test_security_fixes.py` §1 |
| `/info/debug/teams_with_games` admin guard | `test_security_fixes.py` §2 |
| OTP / password-reset rate limit | `test_security_fixes.py` §3 |
| `orchestrator_v3` dangling `count` reference | `test_critical_bug_fixes.py` Bug 1 (AST) |
| `postmatch_refresh.SeasonNames.from_date()` | `test_critical_bug_fixes.py` Bug 2 |
| `transaction_service.create_manual_withdrawal` static→instance | `test_critical_bug_fixes.py` Bug 3 (AST) |
| `BetNotificationTracking` FK target | `test_critical_bug_fixes.py` Bug 4 |
| Admin SF enrich/onboard/status routes | `flow_tests/admin/test_sofascore_routes.py` |
| `/info/sofascore/*` data routes | `flow_tests/sofascore/test_sf_data_routes.py` |
| End-to-end SF onboarding | `flow_tests/sofascore/test_onboard_flow.py` |
| Multi-stage SF coverage flow | `flow_tests/sofascore/test_sf_coverage_e2e.py` (632 LOC) |
| `LeagueOnboarder` | `tests/unit/sofascore/test_league_onboard.py`, `test_league_onboard_service.py`, `tests/unit/admin/test_league_onboard_readiness.py` |
| `SofascoreIdResolver` | `test_sofascore_id_resolver.py` |
| `season_parser`, field mapping, schema drift, name matcher | individual `tests/unit/sofascore/test_*.py` files |
| `match_state` SF lifecycle | `test_match_state.py`, `test_match_state_integration.py` |
| `SingleInstanceLock` | `tests/unit/whoscored/test_orchestrator_v3_lock.py` |
| WS error taxonomy | `tests/unit/whoscored/test_errors.py` |
| WS player resolver | `tests/unit/whoscored/test_player_resolver.py` |
| WS audit | `tests/unit/audit/test_whoscored_audit.py` |
| `match_sink` player dedup | `tests/unit/sinks/test_match_sink_player_dedup.py` |
| WH status normalization | `tests/unit/sinks/test_wh_status_normalization.py` |
| Provider-mapping consolidation | `tests/unit/mappings/test_consolidate_provider_mappings.py` |
| Cross-service BS↔BO sync | `tests/qa_test/odds-pricing-main/layer{1-6}/*.sh` (operational) |

Detail: see [`tests-unit.md`](pr-668-leaves/tests-unit.md) §B and [`tests-integration.md`](pr-668-leaves/tests-integration.md) §A.

## 3. Coverage gaps (prioritized)

From [`tests-coverage-gaps.md`](pr-668-leaves/tests-coverage-gaps.md):

1. **Kafka emit verification** — `KafkaProducer.produce` happy path, `_delivery_callback`, both `_emit_*_event` call sites (`OddsClient.get_parlay_odds`, `UserBetService` post-confirm). Fire-and-forget hides regressions.
2. **`odds_service_client` end-to-end** — `sync_lineup_updates_to_odds_service`, `_post_targeted_migration`, `_fetch_recently_scraped_wh_game_ids`, both `http` and `subprocess` modes. Only the `qa_test/odds-pricing-main/*.sh` cover, and those are not pytest.
3. **`/admin/sofascore/onboard` action paths** — only STATUS covered. LINK / LINK_ALL / DISCOVER / CREATE untested at route level.
4. **Response-model enforcement on `/info/sofascore/*`** — 32 routes return raw dicts despite schemas existing.
5. **`/admin/leagues/onboard*` flow registry** — 5 routes missing from `ROUTE_TO_FLOW`. No drift test in CI.
6. **`prelive_lineups_imminent` 90s tier** — argument shape, `enable_slack=False`, post-step sync hook not unit-tested.
7. **`sf_enrich --tier {…}` selection** — only constants are asserted; runtime tier-to-pipeline-set mapping is not exercised.
8. **`20260420_dedup_competitions_add_unique` migration** — destructive; no upgrade-downgrade-upgrade fixture test.
9. **`WITHDRAWAL_COMPLETED` removal safety** — no test asserts no producer still writes the type or no consumer relies on it.
10. **Background-thread ContextVar propagation** — daemon threads in `_run_sf_*_sync` lose `request_id_ctx` / `flow_ctx`. No test asserts log fields.
11. **`_odds_admin_contracts` shape drift** — hand-mirrored TypedDict; no automated drift detection vs Backend-Odds.
12. **Audit-row write happy paths** — `scrape_runs`, `enrichment_runs`, `whoscored_runs` rows; column shape not asserted.

## 4. Quality concerns

- AST-based bug-fix tests catch regression-by-revert but not regression-by-different-code-path.
- 632-LOC `test_sf_coverage_e2e.py` is large and may be slow / flaky on stub creds; needs verification it follows the skip-on-stub pattern from `.claude/rules/flow-tests.md`.
- `tests/qa_test/odds-pricing-main/*.sh` are valuable for manual QA but are NOT regression coverage in CI (not pytest, not picked up by `flow-tests-mac.yml`).
- Mock-heavy unit tests on `_compute_live_status_map` are appropriate per the testing rule (don't mock the database is for integration tests only).

## 5. PR description checklist (per project rule)

All 4 boxes blank, including:

```
[ ] Unit tests added/updated
[ ] Integration tests added/updated
[ ] Manual QA performed
[ ] Edge cases considered
[ ] ⚠️ IMPORTANT: Backtest with previous dev launch ⚠️
```

Per `.claude/rules/flow-tests.md`, a green local flow-test runner output should be pasted into the PR description. Currently absent.

## 6. Risk shortlist for the merge gate

If the PR can address only ~3 gaps before merge, prioritize:

1. Add `/admin/leagues/onboard*` to `ROUTE_TO_FLOW` and verify `./scripts/check-flow-drift.sh` passes.
2. Fixture-DB upgrade-downgrade-upgrade test for `20260420_dedup_competitions_add_unique`.
3. Wire `response_model=` on at least the highest-traffic `/info/sofascore/*` routes (match data, player profile, team performance).
