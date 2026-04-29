# PR #140 — Pipeline / Service Flows

Inner pipeline+service flows triggered by HTTP/WS/scheduled entries. Branch `dev-coverage-and-odds-impro` → `dev`.

## Summary
- New service flows: 8 (P-CAL, P-PLAYER-CAL, P-CLUSTER, P-MARKET, P-REFIT, P-EXP, P-LINEUP, P-SIM-CAL).
- Modified pipeline branches: 6 (P-FIT, P-ZAP, P-ZAP-BATCH, P-PARLAY, P-TEAMPROP/ZAT, P-LIVE).
- New `@traced` spans: 1 (`simulation.simulate_match_calibrated` `simulation.py:43`).

## Flow walks

### P-FIT: pipeline-startup fit + market calibration (MODIFIED)
- Trigger: DI `ws_dependencies.py:286` per group; also from P-REFIT.
- `MainPredictionPipeline.fit_models` `main_pipeline.py:725` (`@traced pipeline.fit_models`) → `player_pipeline.fit_model` `player_action_pipeline.py:385` → `team_pipeline.fit_model` `team_model_pipeline.py:386`. If `use_market_calibration`, ordered loop over `calibration_order=[h2h,totals,spreads]` calling `team_pipeline.calibrate_with_market_odds/_totals_odds/_spreads_odds` `:768-1020`; per-game errors > `max_game_calibration_error` populate `_rejected_game_ids` (consumed at request time). NEW: `_fit_cluster_model` `:1471` and `_fit_lineup_model` `:1396` (Redis-cached). NEW: `calibrate_player_props` `:1118` (binomial-error gradient against player model).

### P-ZAP: single-leg ZAP probability (MODIFIED)
- Trigger: F12 → `pipeline.get_prob_for_zap` `main_pipeline.py:2024` (`@traced pipeline.get_prob_for_zap`).
- (1) Redis GET `pricing_block:{gid}:{pid}:{action}` `:2065-89` (extreme-policy sentinel; if set return `None`). (2) `PricingContext(self, gid)` `:2119` + `_ensure_adj()` once → loads `MatchCalibrationAdjustment` from `match_cal_adj:{gid}` via `_get_match_calibration` `:237` (TTL-cached). (3) `player_prob_fingerprint(adj, pid, action)` mixed into key (URA-T19). (4) GET `prob:zap:...:adj=<fp>` (TTL ≈15min). (5) Miss → `_sample_player_counts` `:2252`: team_totals from P-SIM-CAL × `player_pipeline.model.predict` × `zonal_prob` × cluster blend × `lineup_weight=expected_minutes/90`. (6) `rng.binomial(n=team_totals, p=effective_rate)` vs line.

### P-ZAP-BATCH: builder all-ZAP (MODIFIED)
- BuilderService → `pipeline.get_all_zap_odds_for_game` `batch_zap.py:690` (`@traced batch_zap.get_all_zap_odds_for_game`). Cache key `builder:all_zap:...:adj={full_adj_fingerprint}` `:719` (URA-T33). `_ensure_models_loaded` gate `:730` (URA-T30: returns `[]`+WARN if unfit, vs prior silent simulate). Single `simulate_match_calibrated(gid, n_simulations=3000, actions=all)` `:770` (was `simulate_match`, URA-T17), vectorized eval over `(player, action, line)`.

### P-PARLAY: parlay leg pricing (MODIFIED)
- Trigger: F10/F11 → `GetOddsForBet.get_odds_for_parlay` `odds_service.py:1416` → `_get_pipeline_parlay_odds` `:1474` → `pipeline.get_odds_for_parlay_bet` `parlay_odds.py:1592` (`@traced parlay.get_odds_for_parlay_bet`).
- `parlay_validator.validate(game_ids)` `live_policy.py:383` → reject ended → split live vs pre-match `odds_service.py:1430-50`. Live → `_get_live_parlay_odds` (matches picks against `live_odds:{gid}` snapshot). Pre-match: build one `PricingContext` per game_id `parlay_odds.py:1634-55` (URA-T4, every leg shares one adj snapshot), embed `full_adj_fingerprint` per game in cache key `:1664` (TTL 30s). Per game `_get_odds_for_match_parlay(bets, gid, ctx=...)` calls `simulate_match_calibrated` (was `simulate_match`) and `PricingContext.get_player_rate` for ZAP legs. Cross-match product clamped at `max_odds_multiplier`.

### P-SIM-CAL: calibrated match simulation (NEW)
- Used by P-ZAP, P-ZAP-BATCH, P-PARLAY, P-TEAMPROP-1.
- `SimulationEngine.simulate_match_calibrated` `simulation.py:43`. Reads adj via `pipeline._get_match_calibration` → `team_lambda_fingerprint(adj)` → key `odds:sim:calibrated:{gid}:{actions}:adj={fp}_v2`. Per-action confidence-blended lambdas `λ_final = c·factor + (1−c)·1.0` (`_apply_match_cal_lambda` `main_pipeline.py:270`) feed `simulate_team_actions_with_lambdas` (vine copula sampling from calibrated marginals — pre-PR fed raw lambdas and post-hoc-multiplied). Cache JSON `CacheTTL.SIMULATION` ≈5min. Distinct namespace from `simulate_match`.

### P-CAL: per-match calibration sweep (NEW)
- Trigger: F19 (`run_match_calibration_check` `tasks.py:1133`, every 120s).
- `MatchCalibrationService.calibrate_match(gid, home, away, trigger_type)` `match_calibration_service.py:549`. Gate on `is_match_calibration_enabled` + `_is_on_cooldown`. Resolve pipeline via injected router. Base lambdas `team_pipeline.model.predict_team_mean(...,"goals")`. `_load_market_moneyline_probs` from Redis `live_odds:{gid}` `:892` (URA-T31: missing key now degrades to identity factors instead of short-circuit). Compute `_divergence(model_probs, market_probs)`; if ≥ `MATCH_CALIBRATION_DIVERGENCE_THRESHOLD` run `_optimise_factors` `:813` (scalar sq-error min); confidence = `max(floor, 1 − residual/divergence)`. Player props (P-PLAYER-CAL) and `_calibrate_team_props` `:1617` (corners/cards μ optimisation) run unconditionally (URA-T27). URA-T27 guard `:701`: if moneyline agrees AND no player adj AND no team adj → `None`. Else build `MatchCalibrationAdjustment(home/away_lambda_factor, confidence, player_adjustments[str_pid][action]=factor, team_adjustments)`; SET `match_cal_adj:{gid}` (TTL ~30min) + cooldown. Optional `pricing_block:*` sentinels via `_apply_extreme_policy` `:294`.

### P-PLAYER-CAL: per-player prop calibration (NEW)
- `_calibrate_player_props` `match_calibration_service.py:1119`. `_load_player_props_from_redis(gid)` `:1512` (URA-T21: Redis-only, never DB). `resolve_unmatched_odds` maps API names → internal `player_id`. Per (player, action): `_get_fitted_r` `:966`, devig market via `devig_power`, infer implied rate, derive multiplicative factor; clamp `[0.3, 3.0]`; warn outside `[0.4, 2.5]` with structured `(pid, action, factor)`. Returns `{str(pid): {action: factor}}`.

### P-TEAMPROP / P-ZAT (MODIFIED)
- F12 → `pipeline.get_odds_for_team_props` `main_pipeline.py:3323` and `pipeline.get_odds_for_zat` `:3373`. `get_probs_for_team_props` `:3104` cached at `team_props:{gid}:{zones}:unified_vig`. NEW analytical fast path `_get_team_props_analytical` `:3038` for full-pitch — Poisson outer-product after `_apply_match_cal_lambda(gid, λ, is_home)` `:3061-62` (~0.1ms vs 374ms sim). Apply `_apply_vig_odds_power`, filter by min/max odds. ZAT: `_get_team_action_rate × zonal_pipeline.get_combined_zonal_probs`; goals action gets `_apply_match_cal_lambda` `:3422-24` before zonal product.

### P-LIVE: live odds read (MODIFIED)
- F12 when `_is_live_priceable` → `_get_live_odds_from_redis` `odds_service.py:449`. Simulate-mode override returns `_get_cached_simulated_odds` `:460-64`. GET `live_odds:{gid}`; missing → `NO_COVERAGE` vs `PENDING` via `_has_historical_odds` (DB). Reject if `age > LIVE_BETTING_POLICY.freshness_threshold_seconds` → `STALE`. `LiveOddsService.get_team_props` `live_odds_service.py:91` reformats moneyline/totals/spreads with power-vig (hold/gamma from pipeline config).

### P-MARKET: batch market-odds blend (NEW)
- F13 → `MarketOddsService.get_market_odds_batch(game_ids)` `market_odds_service.py:155` (`@flow_traced Flow.DATA_INFO`). Pipelined `redis_cache.mget(market_odds:{gid})`; cached entries with `_game_date` invalidate when game crosses `_is_live_window` (5min before kickoff). Misses partition into live/pregame via `_lookup_game_info`. Live → `_fetch_live_kalshi_batch` `:401` reads `live_odds:{gid}`, `_is_payload_fresh`, `_extract_kalshi_from_payload`, `_apply_vig_to_summary`; misses *omitted* (not None) so Backend-Server `if bet_type in odds_data` doesn't TypeError. Pregame → `_compute_analytical(pipeline, gid, home, away)` `:552` (Poisson); cache write embeds `_game_date`. Live entries NOT written to `market_odds:*` (live_odds:{gid} canonical). `MarketOddsApiService` (`market_odds_api_service.py:83`) — DTOs used but no caller.

### P-CLUSTER (NEW)
- Fit (P-FIT step): `PlayerClusterFitter.fit_all_players_with_priors` `cluster_models/fitter.py:254` over `zonal_repo` → `AllPlayerClusters` JSON cached `cluster_model:per_player_params` (TTL ~3600s).
- Blend (P-ZAP step): `_get_cluster_action_rate` `main_pipeline.py:1790` returns `(rate, variance, confidence, source∈{confirmed,predicted,missing,history})` via `PlayerClusterPredictor.predict_from_stats_with_lineup` `cluster_models/predictor.py:565`. In `_sample_player_counts` `:2376-2433`: alpha = `cluster_model_lineup_alpha` (0.7) for confirmed/predicted/missing else `cluster_model_blend_alpha` (0.3); `effective = (1−α)·zonal·rate + α·min(cluster_rate·zonal, 1)`. Variance cached in `_cluster_variance_cache[pid]` for vig widening.

### P-REFIT: blue-green pipeline refit (NEW)
- F5 → `RefitManager.start_refit(group, force, calibrate, calibrate_player_props)` `refit_manager.py:67`. Per-group `threading.Lock` (lazy via `_locks_guard`); running → 409. Daemon `_refit_worker` → `_run_refit` `:144`: re-read seasons via `pipeline_config_repo.get_seasons_for_group` (Issue 1), build new `MainPipelineConfig` from `current_config.to_dict()` + overrides (Issue 2), `create_pipeline_fn(new_config).fit_models(force_refit=force)` (calls P-FIT). URA-T30: `_ensure_models_loaded()=False` → FAILED, no swap. `_validate_pipeline` `:273` runs `predict_match` smoke on 3 recent games (`home/away ∈ [0.1,5.0]`, `n_teams≥4`). Success → atomic `self.pipelines[group] = new_pipeline` → SWAPPED.

### P-EXP: exposure-snapshot flush (NEW)
- F22 (`exposure_task` 300s, `app.py:325`) → `asyncio.to_thread(ExposureService.flush_snapshots)` `exposure_service.py:143`. `SCAN_ITER match=exposure:match:*` (cluster-safe per-node). Per match `HGETALL` → `(bet_count, total_stake, single_stake, parlay_stake)`; build `ExposureSnapshots` ORM row (var95/var99 placeholder 0.0); bulk INSERT.

### P-LINEUP: lineup-change detection (NEW)
- F18 per game → `LineupChangeDetector.detect(new_lineup)` `lineup_change_detector.py:139`. GET Redis `lineup:{gid}` (24h TTL); set-diff over starters+bench (starter↔bench swaps count); on diff SET cache, return `(diff, True)` for caller to upsert + Kafka publish.

## Cross-cutting

- `PricingContext` (`pricing_context.py`) snapshots adj exactly once per (pipeline, game), threaded through all three ZAP paths + parlay — closes URA-T17/T19 staleness class.
- `cache_fingerprint.py` (`team_lambda_fingerprint`, `player_prob_fingerprint`, `full_adj_fingerprint`) mixes only the adj slice each cache reads, keeping hit rate while preventing mid-window staleness.
- `vig_policy.apply_power_vig` is the new single source for power-vig math (consumed by `MarketOddsService`, `GetOddsForBet`, `MainPredictionPipeline._apply_vig_odds_power`); deduplicates three prior copies.
