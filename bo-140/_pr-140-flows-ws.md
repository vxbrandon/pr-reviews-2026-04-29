# PR #140 — WebSocket Flows

Branch: `dev-coverage-and-odds-impro` → `dev`. Live WS endpoint: `/ws/{client_id}` (`infrastructure/api/websocket/ws_routes.py:14`).

## Summary
- New subscription channels: 0
- New server push types: 0
- Protocol-breaking changes: 1 (docs-only example field rename `zao` → `zao_bets`; verified no code emits a top-level `zao` key)

PR #140 changes only **import paths** inside the WS module (service package re-org). No connection-lifecycle, message-routing, span, or schema code was modified. The WS subscription/broadcast flow listed as F14 in `_pr-140-flows-v1.md` (run_odds_broadcast loop) is unchanged behaviorally.

## Connection lifecycle
- No changes in PR #140. Existing flow (for context):
  - `ConnectionManager.connect` opens `tracer.start_as_current_span("websocket.connect")` and seeds connection state with `spotlight_subscribed=False`, `subscribed_season_ids=[]`, `subscribed_season_names=[]` (`manager.py:70-81`).
  - `ConnectionManager.disconnect` wraps cleanup in `websocket.disconnect` span (`manager.py:100`).
  - No auth on `/ws/{client_id}` — `client_id` is taken straight from URL path (`ws_routes.py:14`).
  - Ping/pong: client sends `{"action":"ping"}`; server replies `{"type":"pong","timestamp":"..."}` (`handler.py:185-187`, `handler.py:335`). No server-initiated heartbeat.

## Subscription channels (subscribe / unsubscribe)
Routed by `action` field in `WebSocketHandler.handle_message` (`handler.py:152-191`). All five branches existed pre-PR; PR did not touch this dispatch.

| Channel name | Handler | file:line | Status | Service called |
|---|---|---|---|---|
| `subscribe` (game odds: zap/zat/zao/team_prop/mom) | `WebSocketHandler.handle_subscribe` | `handler.py:213` | unchanged | `ConnectionManager.update_subscription` `manager.py:518` → `GetOddsForBet` (`application.service.odds.odds_service`) |
| `subscribe` (`is_builder` / `is_builder_leaders` branch) | `WebSocketHandler.handle_builder_subscribe` | `handler.py:256` | unchanged | `BuilderService.generate_quick_picks` (`application.service.odds.builder_service`) — import path renamed `handler.py:26` |
| `unsubscribe` | `WebSocketHandler.handle_unsubscribe` | `handler.py:314` | unchanged | `ConnectionManager.remove_subscription`; replies `{type:"unsubscribed", game_id}` `manager.py` / `handler.py:325` |
| `subscribe_spotlight` | `WebSocketHandler.handle_subscribe_spotlight` | `handler.py:338` | unchanged | `ConnectionManager.subscribe_spotlight` `manager.py:968` (uses `manager.odd_service.get_spotlight_odds`) |
| `unsubscribe_spotlight` | `WebSocketHandler.handle_unsubscribe_spotlight` | `handler.py:382` | unchanged | `ConnectionManager.unsubscribe_spotlight` `manager.py:1035` |
| `ping` | `WebSocketHandler.handle_ping` | `handler.py:327` | unchanged | none |
| any other `action` value | inline default branch | `handler.py:188-191` | unchanged | replies `{type:"error", message:"Unknown action: ..."}` |

## Server push types
All emitted via `ConnectionManager.send_personal_message` → `websocket.send_json` (`manager.py:122-139`). No new types added in PR #140.

| Message type | Producer | file:line | Triggered by | Status |
|---|---|---|---|---|
| `odds_update` | `ConnectionManager.send_odds_update` | `manager.py:416` | broadcast loop / `update_subscription` `manager.py:377` | unchanged |
| `auto_disconnected` | `ConnectionManager` (broadcast & spotlight paths) | `manager.py:312`, `manager.py:385`, `manager.py:951` | inactivity detector | unchanged |
| `odds_update_error` | broadcast error path | `manager.py:334`, `manager.py:395` | exception during per-client broadcast | unchanged |
| `unsubscribed` | `handle_unsubscribe` | `handler.py:325` | client `unsubscribe` action | unchanged |
| `pong` | `handle_ping` | `handler.py:335` | client `ping` action | unchanged |
| `subscription_error` | `handle_subscribe` except | `handler.py:254` | bad subscribe payload / missing `game_id` | unchanged |
| `builder_error` | `handle_builder_subscribe` errors | `handler.py:267, 273, 278, 311` | invalid `zone_group` / `zones` / exception | unchanged |
| `spotlight_subscribed` / `spotlight_unsubscribed` | `handle_subscribe_spotlight` / `..._unsubscribe` | `handler.py:368`, `handler.py:393` | spotlight ops | unchanged |
| `error` (generic) | dispatcher / spotlight fallthrough | `handler.py:190, 200, 374, 397` | unknown action / handler exception | unchanged |
| `player id or team id not found` | `update_subscription` validation | `manager.py:637` | bad `player_id_or_team_id` | unchanged |
| `subscription_limit` | `update_subscription` cap | `manager.py:659` | per-client subscription cap exceeded | unchanged |
| `odds_error` / `initial_odds_error` | `update_subscription` exception | `manager.py:756`, `manager.py:773` | initial-odds compute fail | unchanged |
| `initial_spotlight_error` | `subscribe_spotlight` exception | `manager.py:1031` | spotlight fetch fail | unchanged |
| (builder push) `builder_quick_picks` payload | `send_builder_odds_update` via `handle_builder_subscribe` | `handler.py:303-307` | builder subscribe | unchanged |

## Manual OTEL spans
Per repo CLAUDE.md, the WS handler creates manual spans. PR #140 adds **none**. Existing spans (verified at PR head):
- `websocket.connection` `handler.py:84`
- `websocket.process_message` `handler.py:157`
- `@traced("websocket.subscribe")` `handler.py:213`
- `@traced("websocket.builder_subscribe")` `handler.py:256`
- `@traced("websocket.unsubscribe")` `handler.py:314`
- `@traced("websocket.ping")` `handler.py:327`
- `@traced("websocket.subscribe_spotlight")` `handler.py:338`
- `@traced("websocket.unsubscribe_spotlight")` `handler.py:382`
- `websocket.connect` / `websocket.disconnect` / `websocket.broadcast` / `websocket.update_subscription` / `websocket.broadcast_spotlight` (`manager.py:70, 100, 166, 518, 920`)

## Cross-cuts to services
PR-touched lines (3 import-only edits):
- `manager.py:14` `GetOddsForBet` import → `application.service.odds.odds_service`
- `manager_typed.py:12` same rename (file dormant, see Risk)
- `handler.py:26` `BuilderService` import → `application.service.odds.builder_service`

## Flow walks

### F-WS-1: Game odds subscribe (unchanged path, illustrating service rename impact)
1. Client → `/ws/{client_id}` `ws_routes.py:14`
2. `ConnectionManager.connect` `manager.py:70`
3. Client sends `{action:"subscribe", game_id, bet_type, ...}`
4. `WebSocketHandler.handle_message` dispatches `handler.py:166`
5. `handle_subscribe` parses `SubscribeRequest`; if `is_builder*` branches to F-WS-2 `handler.py:227`
6. `manager.update_subscription(...)` `manager.py:518` — invokes `GetOddsForBet` (newly under `application.service.odds.odds_service`)
7. Initial odds pushed as `odds_update` `manager.py:416`; ongoing updates via broadcast loop (F14 in flows-v1).

### F-WS-2: Builder subscribe (uses renamed BuilderService import)
1. Same dispatch as F-WS-1 step 4.
2. `handle_builder_subscribe` `handler.py:256` validates `zone_group` (must be `CUSTOM_ZONE_GROUP` with `zones[]` or in `DEFAULT_ZONE_GROUP_CONFIGS`) `handler.py:265-280`.
3. Maps UUID `game_id` via `manager.id_mapping_repo.map_game_id` `handler.py:285-286`.
4. `BuilderService(self.manager.odd_service).generate_quick_picks(...)` in a thread `handler.py:299` — service now resolved from `application.service.odds.builder_service`.
5. `manager.send_builder_odds_update(...)` emits the result with `data={builder_quick_picks: ...}` `handler.py:303-307`.

## Risk note (minor)
- `handler_typed.py` and `manager_typed.py` are not wired (no importers outside themselves; `ws_dependencies` instantiates only `ConnectionManager`/`WebSocketHandler` from the non-typed files). The PR's edit to `manager_typed.py:12` is dormant scaffolding — flag for cleanup but no runtime impact.
- Docs-only example field rename in `docs/api/websocket_subscription_api.md:363` (`"zao":` → `"zao_bets":`, plus added `"metadata"` example). The runtime `odds_update` payload is built by `manager.py` / `ConnectionManager.send_odds_update` (`manager.py:416`) and was not modified, so this looks like a doc-fixup to match what the code already emits — but worth a manual confirm against the actual `OddsUpdate` schema before treating as protocol-stable.
