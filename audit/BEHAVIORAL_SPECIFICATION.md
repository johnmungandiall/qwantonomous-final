# Complete Behavioral Specification — qwantonomous

**Method:** `kbcodedev/business-logic-workflow-extractor` v2.1.0, Phases 1–5, run against the
project at the repository root. Every finding carries a provenance pointer and a confidence tag:

- `verified_code` — read directly in the implementation (or an equivalent runtime artefact).
- `inferred_convention` — reasoned from what was read, not itself observed.
- `unverified_gap` — not verifiable from the sources inspected; recorded so it is not mistaken for fact.

**Evidence precedence applied:** live tests/CI (`Tier 1`) > active configuration and env
(`Tier 2`) > code (`Tier 3`) > declarative schema/DI (`Tier 4`) > static defaults (`Tier 5`) >
docs/comments (`Tier 6`). Where a lower tier contradicted a higher one, the higher tier is
reported as the behaviour and the conflict is listed under *Documentation Contradictions*.

**Scope note:** this is a static behavioural extraction, not a live-instrumented measurement.
No latency figure below is a measured p99; where the skill's contract asks for p99 the value is
`null` with a source of `unknown`.

---

## Summary

| # | Metric | Count |
|---|---|---|
| 1 | Backend source files scanned (Python, project packages) | ~1,100 (2,440 tracked files total incl. frontend/docs/brokers) |
| 2 | Entry points catalogued | 3 classes (HTTP web, HTTP API, WebSocket) + 2 outbound-client families + 8 scheduled job groups |
| 3 | Business rules extracted | 28 |
| 4 | End-to-end flows traced | 7 |
| 5 | State machines | 4 |
| 6 | Temporal mechanics (delay / timeout / cron) | 34 |
| 7 | External dependencies | 20 broker families + 8 infrastructure services |
| 8 | Configuration keys (env) | 90+ (from `.sample.env`, fully read) |
| 9 | Feature flags | 13 |
| 10 | Documentation contradictions | 8 |
| 11 | Unknown / unverified behaviours | 15 |
| 12 | Migrations in the ordered runner | 23 |

Runtime platform: **Python >= 3.12**, **Flask 3.1.3** app-factory + **Flask-SocketIO 5.6.1**,
**flask-restx 1.3.2**, **Flask-SQLAlchemy 3.1.1 + SQLAlchemy 2.0.51**, **APScheduler 3.11.2**,
**SQLite (WAL)** as the primary store with **DuckDB** for historical data, **ZeroMQ (pyzmq 27.1.0)**
as an internal tick bus, **httpx[http2]** for broker REST. Product version `2.0.2.1`
(`pyproject.toml#version`). Provenance: `pyproject.toml`, `app.py#create_app`.

---

## Plane A — Domain Rules & Business Invariants

### 1. Business Rules

| ID | Domain | Rule | Condition | Violation action | Source | Confidence |
|---|---|---|---|---|---|---|
| BR-001 | Auth/startup | `APP_KEY` must exist and be >= 32 chars | App boot | `RuntimeError`, app refuses to start | `app.py#create_app` | verified_code |
| BR-002 | Security | `API_KEY_PEPPER` must exist and be >= 32 chars | Module import of auth layer | `RuntimeError` at import | `database/auth_db.py` | verified_code |
| BR-003 | Security | `FLASK_DEBUG=True` is refused on a non-loopback host | `__main__` start, host not in `{127.0.0.1, localhost, ::1, ""}` | writes refusal to stderr, `sys.exit(1)` | `app.py`, module `__main__` block | verified_code |
| BR-004 | Security | Remote MCP must never run with `FLASK_DEBUG` truthy | `MCP_HTTP_ENABLED=True` and debug in `("true","1","t")` | `RuntimeError` at boot | `app.py#create_app` | verified_code |
| BR-005 | Security | `MCP_PUBLIC_URL` is mandatory when MCP HTTP is on | `MCP_HTTP_ENABLED=True` and URL empty | `RuntimeError` at boot | `app.py#create_app` | verified_code |
| BR-006 | Order entry | Required order fields must all be present | `REQUIRED_ORDER_FIELDS` minus keys ≠ ∅ | `400` with "Missing mandatory field(s)" | `services/place_order_service.py#validate_order_data` | verified_code |
| BR-007 | Order entry | `exchange`, `action`, `price_type`, `product` must be in the allowed sets | value not in `VALID_*` | `400`, message names the allowed set | `utils/constants.py`, `services/place_order_service.py#validate_order_data` | verified_code |
| BR-008 | Order entry | `action` is normalised to upper case before validation | always | mixed case accepted | `services/place_order_service.py#validate_order_data` | verified_code |
| BR-009 | Order routing | `semi_auto` users have orders queued, not executed | `get_order_mode(user) == "semi_auto"` | order written to pending store, `200` "queued for approval" | `services/order_router_service.py#should_route_to_pending` | verified_code |
| BR-010 | Order routing | A fixed operation set never queues | `api_type` in `IMMEDIATE_EXECUTION_OPERATIONS` | executes immediately | `services/order_router_service.py#IMMEDIATE_EXECUTION_OPERATIONS` | verified_code |
| BR-011 | Order routing | Routing failure defaults to immediate execution | exception in the routing check | logs and returns `False` (auto) | `services/order_router_service.py#should_route_to_pending` | verified_code |
| BR-012 | Sandbox | Order status is constrained to five values | DB insert/update | `CHECK` constraint rejects | `database/sandbox_db.py#SandboxOrders` | verified_code |
| BR-013 | Sandbox | `action` constrained to `BUY`/`SELL`; `price_type` to `MARKET`/`LIMIT`/`SL`/`SL-M`; `product` to `CNC`/`NRML`/`MIS` | DB write | `CHECK` constraint rejects | `database/sandbox_db.py#SandboxOrders` | verified_code |
| BR-014 | Sandbox | One position row per (user, symbol, exchange, product) | insert duplicate | `UniqueConstraint` rejects | `database/sandbox_db.py#SandboxPositions` | verified_code |
| BR-015 | Sandbox fills | A quote whose `ltp` falls outside its own `[low, high]` is treated as stale and the fill is deferred | `ltp>0 and high>0 and low>0` and `not (low<=ltp<=high)` | order stays pending until a coherent quote | `sandbox/execution_engine.py#quote_looks_stale` | verified_code |
| BR-016 | Sandbox fills | Missing/zero OHLC is never treated as stale | `high<=0 or low<=0` | returns False (fill allowed) | `sandbox/execution_engine.py#quote_looks_stale` | verified_code |
| BR-017 | Session | A session is invalid past the daily boundary if it logged in before it | `now > boundary` and `login_time < boundary` | session invalidated, tokens revoked | `utils/session.py#is_session_valid` | verified_code |
| BR-018 | Session | Daily auto-expiry does not revoke a token another device refreshed after the boundary | a different active session logged in at/after the boundary | only the stale device's row is removed | `utils/session.py#revoke_user_tokens` | verified_code |
| BR-019 | Session | Crypto/24-7 installs never auto-expire | `DISABLE_SESSION_EXPIRY=true` | expiry returns a 365-day TTL | `utils/session.py#is_session_expiry_disabled` | verified_code |
| BR-020 | Session | A trading-session date runs boundary-to-boundary, not midnight-to-midnight | IST now < today's boundary | returns the previous calendar date | `utils/session.py#get_trading_session_date` | verified_code |
| BR-021 | IP security | The IP ban list is enforced twice — WSGI layer and per-route decorator | client IP present in ban list | `403` "Access Denied" before any app logic | `utils/security_middleware.py#SecurityMiddleware`, `#check_ip_ban` | verified_code |
| BR-022 | 404 anti-enumeration | 404s are only tracked for unauthenticated, non-crawler paths | not logged in and path not in safe prefixes | IP may accumulate strikes toward a ban | `app.py`, 404 errorhandler | verified_code |
| BR-023 | Order entry (serializer) | A fractional quantity is rejected for every non-crypto exchange | `exchange not in CRYPTO_EXCHANGES` and `quantity != int(quantity)` | `ValidationError` "Fractional quantity ... is not allowed" | `restx_api/schemas.py#_coerce_quantity_to_int` | verified_code |
| BR-024 | Order entry (serializer) | `quantity` must be strictly > 0; price/trigger/disclosed must be >= 0 | range validators on every order schema | `ValidationError` with the named field | `restx_api/schemas.py#OrderSchema` | verified_code |
| BR-025 | GTT entry (serializer) | OCO requires all four leg prices and `triggerprice_sl < triggerprice_tg`; SINGLE requires exactly one positive trigger | `trigger_type` = OCO / SINGLE | `ValidationError` naming the offending leg | `restx_api/schemas.py#_validate_gtt_place_request` | verified_code |
| BR-026 | GTT / options entry | GTT allows only `CNC`/`NRML` (no MIS); options allow only `MIS`/`NRML` | `product` outside the per-schema set | `ValidationError` explaining MIS is intraday-only | `restx_api/schemas.py#PlaceGTTOrderSchema`, `#OptionsOrderSchema` | verified_code |
| BR-027 | Batching bounds | Options multi-order accepts 1-20 legs; margin calculator accepts 1-50 positions | list length outside the bound | `ValidationError` on the list field | `restx_api/schemas.py#OptionsMultiOrderSchema`, `#MarginCalculatorSchema` | verified_code |
| BR-028 | Market-data sanity | A tick implying a > 20% price move, or older than 60 s, is flagged rather than trusted | `MAX_PRICE_CHANGE_PERCENT = 20.0`, `MAX_DATA_AGE_SECONDS = 60` | validation warning recorded for the tick | `services/market_data_service.py#MarketDataValidator` | verified_code |

**Plane-A summary.** The domain is *broker-agnostic trade order routing and accounting*. The
hard invariants are (a) startup fail-fast on missing/short secrets, (b) an allow-list contract on
order fields with a reference-data-consistent enumeration of exchanges/products/price types, (c)
a two-mode order gate (`auto` executes, `semi_auto` queues behind human approval, with a named
never-queue set), (d) a DB-level enumeration of sandbox order states and uniqueness of positions,
(e) a self-consistency guard on fill prices, and (f) a daily 03:00-IST session/token rollover with a
deliberate multi-device exception.

---

### 2. Entry Points

All ingress is HTTP/WebSocket on one Flask process, plus scheduled job groups in-process.

| ID | Protocol | Identifier | Handler | Auth scope |
|---|---|---|---|---|
| EP-001 | HTTP (Web UI) | React SPA service routes, registered when `frontend/dist` exists | `blueprints/react_app.py#react_bp` | Session cookie; `check_session_expiry` before_request |
| EP-002 | HTTP (REST API) | `/api/v1/*` — ~50 namespaces (`/placeorder`, `/placesmartorder`, `/modifyorder`, `/cancelorder`, `/closeposition`, `/cancelallorder`, `/quotes`, `/multiquotes`, `/history`, `/portfolio`, `/sip`, `/depth`, `/optionchain`, `/intervals`, `/funds`, `/orderbook`, `/tradebook`, `/positionbook`, `/holdings`, `/basketorder`, `/splitorder`, `/orderstatus`, `/openposition`, `/ticker`, `/symbol`, `/search`, `/expiry`, `/optionsymbol`, `/optionsorder`, `/optionsmultiorder`, `/optiongreeks`, `/multioptiongreeks`, `/syntheticfuture`, `/analyzer`, `/ping`, `/telegram`, `/whatsapp`, `/margin`, `/instruments`, `/chart`, `/market/holidays`, `/market/timings`, `/pnl`, `/placegttorder`, `/modifygttorder`, `/cancelgttorder`, `/gttorderbook`) | `restx_api/*` namespaces; type handler e.g. `restx_api/place_order.py#PlaceOrder` | API key in body; CSRF-exempt; per-route Flask-Limiter |
| EP-003 | HTTP (webhooks) | `/chartink/webhook`, strategy webhook, `/flow/webhook/...`, `/postback/...` (broker postback) | `blueprints/chartink.py#webhook`, `blueprints/strategy.py#webhook`, `blueprints/flow.py#trigger_webhook[_with_symbol]`, `blueprints/postback.py#broker_postback` | CSRF-exempt by design; validated in-handler |
| EP-004 | HTTP (OAuth callbacks) | `/auth/broker/...` broker callbacks, `/brlogin/...` | `blueprints/brlogin.py#broker_callback` | CSRF-exempt; provider-driven |
| EP-005 | HTTP (health) | `/health/...` simple and detailed | `blueprints/health.py#simple_health`, `#detailed_health_check` | CSRF-exempt, intended for ELB/K8s probes |
| EP-006 | HTTP (Remote MCP) | `/mcp`, `/mcp/sse`, `/oauth/*`, `/.well-known/*` — registered only when `MCP_HTTP_ENABLED=True` | `blueprints/mcp_http.py#mcp_dispatch`, `#mcp_sse`; `blueprints/mcp_oauth.py` | Bearer token (RS256) + PKCE/ZCR; CSRF-exempt on 5 named endpoints |
| EP-007 | WebSocket (market data + orders) | `ws://WEBSOCKET_HOST:WEBSOCKET_PORT` (client-facing), ZMQ SUB bound to `ZMQ_HOST:ZMQ_PORT` (internal) | `websocket_proxy/server.py#WebSocketProxy.handle_client`, internal `#zmq_listener` | Client must `authenticate` with an API key within `WS_AUTH_GRACE_SECONDS` |
| EP-008 | Socket.IO (browser push) | Default Socket.IO path mounted on the Flask app | `extensions.py#socketio` consumers in `subscribers/socketio_subscriber.py` | Flask session |
| EP-009 | Scheduled (APScheduler) | 8 job groups: python-strategy cron, strategy cron, chartink schedule, sandbox square-off/backup/T+1/reset/PnL, Flow jobs, Historify jobs | see *Temporal Mechanics* | in-process, no auth |
| EP-010 | Background daemons | health sampler, latency monitor, broker keep-warm, order-update adapters boot scan, cache restoration, DB init + schedulers | `app.py#setup_environment`, `utils/health_monitor.py`, `utils/latency_monitor.py`, `services/broker_keepalive_service.py`, `services/order_update_service.py#start_order_update_adapters_on_boot` | in-process |

**Confidence:** `verified_code` for the blueprints/namespace/route registrations
(`app.py#create_app`, `restx_api/__init__.py`); the WebSocket client-facing default port is an
`inferred_convention` where it comes from the library default rather than a literal (see
*Documentation Contradictions* DC-004).

---

### 3. Call Graph (illustrative edges, not exhaustive)

| From | To | Invocation type |
|---|---|---|
| `app.py#create_app` | `utils/security_middleware.py#init_security_middleware` | synchronous (wraps `wsgi_app`) |
| `app.py#create_app` | `utils/traffic_logger.py#init_traffic_logging` | synchronous (wraps `wsgi_app` again) |
| `app.py#create_app` | `subscribers/__init__.py#register_all` | synchronous |
| `app.py#create_app` | `utils/event_bus.py#EventBus.subscribe` (via `register_all`) | synchronous |
| `restx_api/place_order.py#PlaceOrder.post` | `services/place_order_service.py#place_order` | synchronous |
| `services/place_order_service.py#place_order` | `services/order_router_service.py#should_route_to_pending` | synchronous |
| `services/place_order_service.py#place_order` | `restx_api/schemas.py#OrderSchema.load` (via `validate_order_data`) | synchronous |
| `services/place_order_service.py#place_order` | `services/sandbox_service.py#sandbox_place_order` | synchronous (analyze mode) |
| `services/place_order_service.py#import_broker_module` | `broker.<name>.api.order_api` | synchronous dynamic import |
| `services/place_order_service.py#place_order_with_auth` | `utils/event_bus.py#EventBus.publish` | async dispatch (thread-pool) |
| `utils/event_bus.py#EventBus.publish` | `subscribers/{log,socketio,telegram,whatsapp,wsproxy}_subscriber` | async dispatch |
| `sandbox/execution_engine.py#ExecutionEngine.check_and_execute_pending_orders` | `services/quotes_service.py#get_multiquotes` | synchronous |
| `sandbox/execution_engine.py#ExecutionEngine.check_and_execute_pending_orders` | `utils/event_bus.py#EventBus.publish` | async dispatch |
| `services/order_update_service.py#_build_adapter` | `broker.<name>.streaming.<name>_order_adapter` | synchronous dynamic import |
| `websocket_proxy/server.py#WebSocketProxy.zmq_listener` | `(client websockets)` | async send |
| `database/auth_db.py#upsert_auth` | `services/order_update_service.py#start/stop_order_update_adapter` | synchronous (lifecycle hook) |

**Confidence:** `verified_code` for the edges read in the named functions; the full graph is not
enumerated (see *Unknown/Unverified* U-03).

---

### 4. End-to-End Flows

**FLOW-001 — Live order placement (happy path).** `verified_code`.
1. `POST /api/v1/placeorder/` → `restx_api/place_order.py#PlaceOrder.post`; Flask-Limiter applies `ORDER_RATE_LIMIT` (default `10 per second`).
2. Marshmallow `OrderSchema.load` validates and deserialises; failure → `400`.
3. `services/place_order_service.py#place_order`; if `api_key` present and no direct (token, broker), consult the routing gate.
4. `should_route_to_pending` → if `semi_auto`, `queue_order` writes the pending row and emits `pending_order_created`; flow ends (`200`, `mode=semi_auto`).
5. Otherwise `validate_order_data` runs the required-field and enum checks.
6. `get_auth_token_broker(api_key)` resolves the encrypted broker token; unknown key → `403`.
7. If analyze mode is on → `sandbox_place_order` (paper path, FLOW-005).
8. Else dynamic-import `broker.<name>.api.order_api` → `place_order_api(order_data, auth_token)`.
9. On `res.status == 200` → publish `OrderPlacedEvent`; else publish `OrderFailedEvent`.
10. EventBus dispatches to log / socketio / telegram / whatsapp subscribers asynchronously.

- **Transaction boundary:** no cross-service transaction; the pending store write (step 4) is the only DB mutation before the broker call, and the broker call is outside it.
- **Critical path:** yes.

**FLOW-002 — Broker authentication / login.** `verified_code` for the enumerated steps; the per-broker OAuth internals are `unverified_gap`.
1. `blueprints/auth.py` login (rate-limited `LOGIN_RATE_LIMIT_MIN`/`_HOUR`); session `login_time` set in IST; `PERMANENT_SESSION_LIFETIME` pinned to the daily boundary.
2. Redirect to the broker's OAuth flow → provider → callback `blueprints/brlogin.py#broker_callback` (CSRF-exempt).
3. Broker auth module resolved lazily via `utils/plugin_loader.py#_LazyBrokerAuthDict.get` → `broker.<name>.api.auth_api#authenticate_broker`.
4. Token persisted encrypted (Fernet/PBKDF2) through `database/auth_db.py#upsert_auth`; `is_revoked=False`.
5. `upsert_auth` triggers the order-update adapter restart/stop (`services/order_update_service.py`).
6. WebSocket feed established for the session.

**FLOW-003 — Market-data tick delivery.** `verified_code` for topology; per-broker parse is `unverified_gap`.
Broker adapter (PUB) → ZMQ bus → `WebSocketProxy` SUB (`tcp://ZMQ_HOST:ZMQ_PORT`) → `subscription_index` lookup by `(symbol, exchange, mode)` → 50 ms throttle → per-client websockets send. Order updates ride the same bus via `order_subscribers`.
- **Bottleneck:** the ZMQ SUB is a single binder and a single listener task in the proxy process; fan-out is one asyncio loop.

**FLOW-004 — Broker order-update ingestion.** `verified_code`.
Two shapes: a dedicated order WebSocket adapter from `_BROKER_FACTORIES` for 16 named brokers, or `PollingOrderUpdateAdapter` for `_POLLING_BROKERS = {groww, fivepaisa, samco, flattrade}`. Both publish `order.update`, which only `socketio_subscriber` and `wsproxy_subscriber` consume (deliberately not logged/alerts — the reasons are documented inline).

**FLOW-005 — Analyzer / sandbox (paper) execution.** `verified_code`.
`sandbox_place_order` → order row; `ExecutionEngineThread` polls every `order_check_interval` (default 5 s) → groups pending orders by symbol → one `get_multiquotes` batch per cycle → per-order fill evaluation (stale-quote guard BR-015, price-type rules) → trade + position updates → EventBus events.

**FLOW-006 — Overnight rollover.** `verified_code`.
`utils/session.py#is_session_valid` is called by the `check_session_expiry` before_request; at the first request after the boundary a stale session is revoked (multi-device guard BR-018), a `force_logout` Socket.IO event broadcasts, and the sandbox PnL reset cron runs at the same boundary.

**FLOW-007 — Data download (historify).** `verified_code` for the scheduler; the downloader internals are `unverified_gap`.
`services/historify_scheduler_service.py#HistorifyScheduler` schedules interval/cron jobs (cron in `Asia/Kolkata`) that call `services/historify_service.py`, which chunks work with `time.sleep(delay)`/`batch_delay` pacing.

---

### 5. State Machines

**SM-001 — Sandbox order lifecycle.** `verified_code`.
States: `open`, `trigger pending`, `complete`, `cancelled`, `rejected` (enforced by the `CHECK` constraint on `database/sandbox_db.py#SandboxOrders`).
Transitions observed in `sandbox/execution_engine.py` and the sandbox services: `open → complete` (marketable fill), `open → trigger pending` (SL/SL-M not yet triggered), `trigger pending → complete | open` (trigger released or price moved away), `open|trigger pending → cancelled` (cancel path), `open → rejected` (validation/margin failure). Partial fills are explicitly not modelled — `filled_quantity` is documented in the schema as "Always 0 or quantity".

**SM-002 — Broker auth-session lifecycle.** `verified_code`.
Fields on `database/auth_db.py#Auth`: `is_revoked` boolean; `auth`/`feed_token` ciphertext. Lifecycle: unauthenticated → authenticated (`is_revoked=False`, token written by `upsert_auth`) → revoked (`is_revoked=True`, by logout or daily auto-expiry) → re-authenticated at next login. A token is treated as stale if no login occurred since the current rollover boundary.

**SM-003 — GTT (Good-Till-Triggered) lifecycle.** `verified_code` for the mechanism; the per-state transitions are `inferred_convention`.
A `sandbox.gtt_manager` owns claim/reclaim/expire; the maintenance thread reclaims stranded claims every `RECLAIM_INTERVAL_SEC=60` and expires due GTTs every `EXPIRY_INTERVAL_SEC=3600`. Event topics exist for `gtt.placed`, `gtt.failed`, `gtt.modified`, `gtt.modify_failed`, `gtt.cancelled`, `gtt.cancel_failed`, `gtt.triggered`, `gtt.expired` (`subscribers/__init__.py`).

**SM-004 — Scalping position stop lifecycle.** `verified_code`.
`services/scalping_risk_monitor_service.py#evaluate_trail` is a pure function over `{side, entry_price, trailing_enabled, trailing_step, target, current_sl, highest/lowest_price}` returning an updated stop and a `breached` flag with reason `sl`/`target`. Long: stop only rises; short: stop only falls; trailing only arms once profit ≥ `MIN_TRAIL_PROFIT`.

---

### 6. Decision Graphs

**DG-001 — Order destination decision.** Inputs: `api_key`, `api_type`, user order mode, analyze-mode flag, availability of a direct (token, broker) pair. Branches: `api_type ∈ IMMEDIATE_EXECUTION_OPERATIONS → immediate`; `order_mode == semi_auto → queue`; `analyze_mode → sandbox`; else → live broker. Source: `services/order_router_service.py#should_route_to_pending`, `services/place_order_service.py#place_order`.

**DG-002 — Fill decision in the sandbox engine.** Inputs: order status, quote availability, `ltp` vs day `[low, high]`, price type, trigger price, order rate limit. Branches: no quote → leave pending (no per-symbol retry, deliberately, to protect the API quota); `quote_looks_stale` → defer; marketable/triggered → fill at the quote; else stay resting.

**DG-003 — Session/token decision at request time.** Inputs: `logged_in`, `login_time`, now vs daily boundary, `DISABLE_SESSION_EXPIRY`, presence of a fresher session. Branches: disabled → always valid; before boundary → valid; after boundary with an older login → invalid; with a fresher sibling session → logout only this device.

**DG-004 — WebSocket handshake acceptance.** Inputs: time since connect, presence of a valid `authenticate` message, banned IP, stale-tick age. Branches: no auth within `WS_AUTH_GRACE_SECONDS` → close; banned → rejected upstream; connected but silent > `WS_STALE_TICK_WARN_SECONDS` → warn only (explicitly never auto-evict).

**DG-005 — Exchange→ZMQ/tick-mode normalisation.** `websocket_proxy/mode_utils.py#normalize_mode` accepts case-insensitive strings and ints and maps to `(int, label)`; `normalize_mode_or_none` returns `None` for unknown. This is the single acceptance point for client-supplied mode values.

---

### 6b. Middleware & Interceptor Chain (and stated absences)

The request path is wrapped, outermost → innermost, in this fixed order
(`app.py#create_app`, in registration order):

| # | Layer | Responsibility | Auth / scoping | Source |
|---|---|---|---|---|
| 1 | `TrafficLoggerMiddleware` | duration, status, IP, method, path, user_id capture; write deferred to a 1-worker executor | attribution only | `utils/traffic_logger.py#TrafficLoggerMiddleware` |
| 2 | `SecurityMiddleware` | WSGI-level ban check; banned IPs get 403 before Flask sees the request | IP allow/deny | `utils/security_middleware.py#SecurityMiddleware` |
| 3 | Socket.IO error handler | uncaught Socket.IO handler errors | n/a | `utils/socketio_error_handler.py#init_socketio_error_handling` |
| 4 | CSP middleware | emits the `CSP_*` response headers | n/a | `csp.py#apply_csp_middleware` |
| 5 | Flask-CORS | origin/method/header allowlists from `CORS_*` | n/a | `cors.py#get_cors_config` |
| 6 | Flask-Limiter | per-route rate limits, moving-window, in-memory store | per client IP | `limiter.py#limiter` |
| 7 | Flask-WTF CSRF | token validation, with a named exemption list | session | `app.py#create_app` |
| 8 | `before_request` hooks | `wait_for_db_ready` (up to 30 s), `check_session_expiry` | session | `app.py#create_app` |
| 9 | Route decorators | `check_session_validity`, `invalidate_session_if_invalid`, `check_ip_ban` | session / IP | `utils/session.py`, `utils/security_middleware.py#check_ip_ban` |

**Authentication layers (three, distinct):** session cookie + Flask-WTF CSRF for the web UI;
API key in the request body for `/api/v1/*` (CSRF-exempt because it is not cookie-authenticated);
RS256 bearer token + PKCE for the optional Remote MCP surface. Broker callbacks and webhooks are
CSRF-exempt by design and validated in-handler.

**Stated absences (searched, none found):**
- **Distributed tracing — none.** A search for `opentelemetry|otel|jaeger|zipkin|tracer|init_tracing`
  across all Python files returns zero matches. Observability is per-request latency
  (`utils/latency_monitor.py`) and traffic rows, not spans. This is a real gap against the skill's
  "distributed tracing" middleware item, recorded rather than omitted.
- **Tenancy scoping — none.** A search for `tenant_id|tenant|multi_tenant|multitenant` returns zero
  matches. The platform is single-install and (per `utils/session.py#is_session_expiry_disabled`'s own
  note) each instance serves one broker. The nearest thing to a scope boundary is the per-`user_id`
  ownership carried on orders, sessions and caches — not an organisational tenant model.

## Plane B — Workflow & Execution Topologies

### 7. Database Behaviour

| Store | Engine / URL | Pooling | Isolation / pragmas | Notes |
|---|---|---|---|---|
| openalgo.db (auth, users, settings, apilog, strategies, flow, telegram, whatsapp, chart prefs, market calendar, qty freeze, Action Center, scalping, leverage, strategy portfolio, APScheduler jobstores) | SQLite `DATABASE_URL` (default `sqlite:///db/openalgo.db`) | **NullPool** forced for SQLite (`database/engine_factory.py#create_db_engine`) | `PRAGMA journal_mode=WAL`, `synchronous=NORMAL`, `busy_timeout=15000` applied per connection (`database/__init__.py`) | Shared by the Flask worker AND the out-of-process WebSocket proxy |
| sandbox.db | SQLite `SANDBOX_DATABASE_URL` | NullPool | inherits the process-wide pragmas | `database/sandbox_db.py` |
| logs.db | SQLite `LOGS_DATABASE_URL` | its own session | inherits pragmas | traffic log + IP ban tables |
| latency.db | SQLite `LATENCY_DATABASE_URL` | its own session | inherits pragmas | latency monitor |
| health.db | SQLite `HEALTH_DATABASE_URL` | its own session | inherits pragmas | health sampler |
| historify.duckdb | DuckDB `HISTORIFY_DATABASE_URL` | n/a | DuckDB default | historical candles |

- **Locking:** no `SELECT ... FOR UPDATE` was found; concurrency is handled with `busy_timeout` plus single-writer discipline. The DB-init phase deliberately runs its 20 table-creation functions on a **`ThreadPoolExecutor(max_workers=1)`** because 14 of them target the same SQLite file and SQLite allows one writer per file (`app.py#setup_environment`, inline rationale).
- **Constraints/triggers:** declarative `CHECK`/`UniqueConstraint`/indexes on sandbox tables; no database triggers were found. `onupdate=func.now()` timestamp maintenance is ORM-side.
- **Sessions:** every request releases its scoped sessions in `app.py#shutdown_database_sessions`; background threads call the same cleanup registry (`utils/db_sessions.py`).
- **Integrity note:** `PRAGMA busy_timeout` does not help a WAL `SQLITE_BUSY_SNAPSHOT`, which reports the same message but returns immediately — documented in `database/__init__.py`.
- **Schema access style:** ORM only. A search for raw `text(` / `execute(` across `database/*.py` returns no matches, so there is no hand-written SQL to audit for injection in the data layer, and there are no stored procedures or triggers to hold hidden rules. Hidden logic lives in ORM column defaults and constraints instead (see section 21).
- **Migration history (Tier 4):** `upgrade/migrate_all.py` runs a fixed, ordered list of **23 idempotent migrations** via subprocess, each independently re-runnable and skipping when already applied. The ordering is load-bearing (core feature migrations before performance/index migrations). `upgrade/rotate_pepper.py` is **deliberately excluded** from that list — it rotates `API_KEY_PEPPER`, re-encrypts every pepper-derived ciphertext and invalidates all Argon2 password hashes, so it is operator-invoked only (`cd upgrade && uv run rotate_pepper.py`) and never run unattended on update. Other migrations of note: `migrate_gtt.py`, `migrate_gtt_sandbox.py` (adds CAS F&O close at 15:40), `add_totp_purpose_flags.py` (per-purpose 2FA), `migrate_indexes.py`, `migrate_zerodha_new_exchanges.py`.
- **Migration failure policy:** a migration returning non-zero is reported as "Completed with warnings" and the run **continues** (the runner returns `True`); a raised exception counts as a failure and is tallied. A missing script is skipped as non-fatal. So the runner is best-effort, not transactional.
- **Confidence:** `verified_code`.

### 8. Cache Behaviour

All caches are in-process `cachetools.TTLCache`; there is **no Redis/memcached tier**. Invalidation is by explicit delete plus TTL backstops.

| Cache | Size | TTL | Source | Invalidation |
|---|---|---|---|---|
| `auth_cache` | 1024 | session-based (min 300 s, max 24 h; until the 03:00 boundary) | `database/auth_db.py#get_session_based_cache_ttl` | deleted in `revoke_user_tokens`; ZMQ `CACHE_INVALIDATE_ALL` to the proxy |
| `feed_token_cache` | 1024 | session-based | same | same |
| `broker_cache` | 1024 | 3000 s | `database/auth_db.py` | broker switch |
| `verified_api_key_cache` | 1024 | 36000 s (10 h) | same | key regeneration |
| `invalid_api_key_cache` | 512 | 300 s | same | anti-poisoning short TTL |
| `order_mode_cache` | 128 | 60 s | same | `update_order_mode` → `invalidate_user_cache` |
| `_settings_cache` | 10 | 3600 s | `database/settings_db.py` | `clear_settings_cache` on logout |
| `_strategy_webhook_cache` / `_user_strategies_cache` | 5000 / 1000 | 300 s / 600 s | `database/strategy_db.py` | `clear_strategy_cache` |
| `_workflow_webhook_cache` / `_workflow_cache` | 5000 / 1000 | 300 s / 600 s | `database/flow_db.py` | workflow mutation |
| telegram caches (user, username, prefs, creds) | 10000 each | 1800 s | `database/telegram_db.py` | `clear_telegram_cache` on logout |
| whatsapp caches | 10000 each | 1800 s | `database/whatsapp_db.py` | logout |
| `_ip_ban_cache` | 2048 | 60 s | `database/traffic_db.py` | ban list mutation |
| `username_cache` | 1024 | 30 s | `database/user_db.py` | user mutation |
| `_stats_cache` (latency) | 1 | 60 s | `database/latency_db.py` | TTL only |
| `_leverage_cache` | 1 | 3600 s | `database/leverage_db.py` | TTL only |
| `_timings_cache` / `_holidays_cache` | 500 / 50 | 3600 s | `database/market_calendar_db.py` | TTL only |
| `_HOLIDAY_CACHE` | 8 | 86400 s | `utils/trading_calendar.py` | TTL only |
| `_history_cache` (indicator) | env `_HISTORY_CACHE_MAXSIZE` | env `_HISTORY_CACHE_TTL` | `services/indicator_service.py` | TTL only |
| funds cache (flattrade) | env `_FUNDS_CACHE_MAXSIZE` | env `_FUNDS_CACHE_TTL` | `broker/flattrade/api/funds.py` | TTL only |
| `_UNDERLYING_NAME_CACHE` | 2048 | 3600 s | `broker/hdfcsecurities/mapping/order_data.py` | TTL only |
| `_ID_CACHE` / `_id_refresh_attempts` | 4096 each | 86400 s / 300 s | `broker/indmoney/streaming/indmoney_order_adapter.py` | TTL only |

- **Master-contract symbol cache** is additionally invalidated on logout via `database/master_contract_cache_hook.py#clear_cache_on_logout`.
- **Pattern:** cache-aside throughout; no write-through. Confidence: `verified_code`.

### 9. External Dependencies

**Broker families (35 in `VALID_BROKERS`, `broker/<name>/` plugins):**
fivepaisa, fivepaisaxts, aliceblue, angel, arrow, compositedge, dhan, dhan_sandbox, definedge,
deltaexchange, firstock, flattrade, fyers, groww, hdfcsecurities, hdfcsky, ibulls, iifl, iiflcapital,
indmoney, jainamxts, kotak, motilal, mstock, nubra, paytm, pocketful, rmoney, samco, shoonya,
tradejini, tradesmart, upstox, wisdom, zebu, zerodha.

Each is a plugin directory with `api/{auth_api,order_api,data,funds,margin_api}.py`, a
`mapping/` transform layer, `plugin.json` capabilities, and a `streaming/` adapter where a push feed
exists. Integration is **direct REST/WebSocket per broker — no vendor SDK**; the plugin contract is
`plugin.json` (`supported_exchanges`, `broker_type`, `leverage_config`) plus the module-level function
names the loader and services import (`utils/plugin_loader.py`, `services/order_update_service.py`).

**Infrastructure services:**
| Service | Integration | Endpoint source | Timeout (observed) | Failure mode |
|---|---|---|---|---|
| Broker REST | httpx HTTP/2 (HTTP/1.1 fallback) | per-broker `baseurl.py` (`ROOT_URL`/`BASE_URL`) | **120 s** client-wide (`utils/httpx_client.py`) | fail to caller; per-broker retry policies |
| Broker WebSocket | per-broker adapter over `websockets`/vendor libs | broker config | per-broker reconnect backoff | reconnect with exponential backoff |
| Telegram | python-telegram-bot 22.8 | bot token in DB | library default | logged, alert dropped |
| WhatsApp | local bot service | DB session | n/a | `409` if not paired |
| ngrok | pyngrok 7.5.1 | `NGROK_ALLOW` | n/a | tunnel absent when disabled |
| MCP hosted clients | HTTP+SSE, RS256 JWT | `MCP_PUBLIC_URL` | access TTL configurable, hard-capped 3600 s | `401`/`403` |
| DuckDB historify | embedded file DB | `HISTORIFY_DATABASE_URL` | n/a | download job failure |
| Chromium/Kaleido | subprocess for chart PNG | image build | CI smoke test asserts PNG > 1000 bytes | render failure logged |

**Confidence:** `verified_code` for the client-wide 120 s timeout, the broker inventory, and the
plugin contract; per-broker timeouts/backoffs are `verified_code` only where read (angel `max_retries=2`
with `0.5 * 2**attempt`, dhan `MAX_RETRIES` on error code 805 with `2.0 * 2**retry`) and
`unverified_gap` for the rest.

### 10. Queue / Event Topology

There is **no external message broker** (no Kafka/RabbitMQ/SQS). Two internal transports:

**T-1 — In-process EventBus** (`utils/event_bus.py#EventBus`). Topic pub/sub, all callbacks on a
shared `ThreadPoolExecutor(max_workers=10)`, `DEFAULT_MAX_PENDING=1000`. Backpressure policy:
**drop-newest with sparse logging** (first drop, then every 100th) rather than block the publisher —
chosen explicitly because subscribers are best-effort side effects and blocking would stall the order path.

Topics observed (`subscribers/__init__.py` — the authoritative subscription map):
`order.placed`, `order.failed`, `order.no_action`, `order.modified`, `order.modify_failed`,
`order.cancelled`, `order.cancel_failed`, `orders.all_cancelled`, `order.update`, `position.closed`,
`basket.completed`, `split.completed`, `options.completed`, `multiorder.completed`, the eight `gtt.*`
topics, `analyzer.error`, `sandbox.order_filled`, `sandbox.auto_squareoff`, `sandbox.t1_settlement`.
Subscriber fan-out per topic is typically log + socketio + telegram + whatsapp; `order.update` and
the sandbox-internal topics deliberately wire only socketio (plus `wsproxy` for `order.update`).

**T-2 — ZeroMQ tick bus** (`websocket_proxy/server.py#WebSocketProxy.__init__`,
`websocket_proxy/connection_manager.py#SharedZmqPublisher`). **Fan-in topology**: the proxy's SUB is
the single *binder* on `tcp://ZMQ_HOST:ZMQ_PORT`; every publisher *connects* to it. `SUBSCRIBE b""`
(receive all topics) because the same bus carries market data **and** `CACHE_INVALIDATE_*` control
messages. `LINGER=1000`, `SNDHWM=1000`. The boot scan warms the publisher early to avoid a
slow-joiner drop of the first order update.
- **Dead-letter queue:** none. Failed work is logged and dropped, or retried in place.
- **Ack/nack:** none (ZMQ PUB/SUB is fire-and-forget; a message with no SUB subscriber is lost).
- **Confidence:** `verified_code`.

### 11. Parallelism

| Scope | Model | Limit | Source |
|---|---|---|---|
| DB table initialisation on boot | `ThreadPoolExecutor` | **1** (deliberate single worker) | `app.py#setup_environment` |
| Analyzer-mode service start on boot | `ThreadPoolExecutor` | 3 (engine, square-off scheduler, catch-up settlement) | `app.py#setup_environment` |
| Event subscriber dispatch | `ThreadPoolExecutor` | 10 workers, 1000 pending cap | `utils/event_bus.py#EventBus.__init__` |
| Traffic-log writes | `ThreadPoolExecutor` | 1 (serialises SQLite writes off the request thread) | `utils/traffic_logger.py` |
| httpx connection pool | keep-alive pool | `max_keepalive_connections=40`, `max_connections=100`, `keepalive_expiry=30 s` | `utils/httpx_client.py#_create_http_client` |
| Broker WS connections per user | connection pool | `MAX_SYMBOLS_PER_WEBSOCKET=1000` × `MAX_WEBSOCKET_CONNECTIONS=3` | `websocket_proxy/connection_manager.py` |
| WS per-client send buffer | asyncio queue | `WS_MAX_QUEUE=1024` | `websocket_proxy/server.py#WebSocketProxy.start` |
| ZMQ publisher | single shared PUB (singleton) | SNDHWM 1000 | `websocket_proxy/connection_manager.py#SharedZmqPublisher` |
| Order/split/options batch chaining | sequential with pacing sleeps | `order_delay` per leg, 1.0 s per basket batch | `services/split_order_service.py`, `services/options_multiorder_service.py`, `services/basket_order_service.py` |

**Confidence:** `verified_code`.

### 12. Concurrency & Race Hazards

- **Thread pools sized to 1 where a shared SQLite file forbids parallelism** — the pattern is applied consistently (DB init, traffic log). `verified_code`.
- **Singleton pattern everywhere concurrency matters:** `EventBus` is a module-level singleton; `SharedZmqPublisher` uses a class-level lock + `_initialized` guard; schedulers (`FlowScheduler`, `HistorifyScheduler`) use `__new__` singletons with a module lock; `ScalpingRiskMonitor` is a singleton guarded by `RLock`; order-update adapters live in a dict guarded by `_LOCK` holding at most one adapter per user.
- **Scalping monitor sets explicit throttles** to avoid races with the browser: `PERSIST_THROTTLE_SEC=1.5`, `EMIT_THROTTLE_SEC=1.0`, `EXIT_RETRY_COOLDOWN_SEC=3.0`, and a coalescing sync worker (the blocking WS subscribe never runs on a request thread).
- **Guard event-loop contamination:** with gunicorn+eventlet the WS proxy is spawned as a **child process, not a thread**, specifically so its asyncio loop never shares an eventlet hub (`websocket_proxy/app_integration.py`, `app.py` inline rationale).
- **Known race the code defends against:** assigning `PERMANENT_SESSION_LIFETIME` at login while a rollover is checked per request; the helper `_todays_rollover_boundary` accepts a clock snapshot so two reads cannot straddle the boundary.
- **`SQLITE_BUSY_SNAPSHOT`** is called out as un-immunisable by `busy_timeout`; the mitigation is to re-run the transaction.
- **Confidence:** `verified_code`.

### 13. Retries & Backoffs

| Target | Max attempts | Policy | Retryable trigger | Source | Confidence |
|---|---|---|---|---|---|
| Strategy-book DB registration at boot | 3 | linear `0.5 * attempt` | any exception (transient SQLite lock) | `app.py#setup_environment` | verified_code |
| Angel `get_api_response` | `max_retries=2` (+1 initial) | exponential `0.5 * 2**attempt`, and chunk retries `1.0 * 2**attempt` up to 4 | rate-limit / transient HTTP | `broker/angel/api/data.py`, `broker/angel/api/order_api.py` | verified_code |
| Dhan `get_api_response` | `MAX_RETRIES` | exponential `RETRY_DELAY(2.0) * 2**retry_count` | error code `805` (rate limit) | `broker/dhan/api/data.py` | verified_code |
| Definedge | per rate-limiter | exponential `BASE_BACKOFF=1.0` → 1,2,4 s, honours `Retry-After` | `429` | `broker/definedge/api/rate_limiter.py` | verified_code |
| Delta Exchange | per rate-limiter | capped wait (`MAX_WAIT_SECONDS`) | `429` | `broker/deltaexchange/api/rate_limiter.py` | verified_code |
| Alice Blue WS | `MAX_RECONNECT_ATTEMPTS` | exponential up to `MAX_RECONNECT_BACKOFF_SECONDS` | connect failure | `broker/aliceblue/api/alicebluewebsocket.py` | verified_code |
| Order-update adapters | continuous | exponential reconnect | `401/403/expired` recognised via `_AUTH_ERROR_INDICATORS` forcing a token re-init | `websocket_proxy/connection_manager.py#_is_auth_error` | verified_code |
| Sandbox GTT | sweep | reclaim every 60 s, expiry every 3600 s | stranded claim past the 60 s claim timeout | `sandbox/execution_thread.py#GTTMaintenanceThread` | verified_code |
| Scalping exit | 1 | retry cooldown 3 s | leg exit keeps failing | `services/scalping_risk_monitor_service.py` | verified_code |
| APScheduler jobs | 1 per tick | `misfire_grace_time=300`, `coalesce=True`, `max_instances=1` | missed/overlapping runs | `sandbox/squareoff_thread.py#start_squareoff_scheduler` | verified_code |

Pacing sleeps that are *not* retries but rate-limit compliance: `services/history_service.py`
(`_MIN_HISTORY_INTERVAL`), `services/oi_profile_service.py` (`BATCH_DELAY`), split/options/basket leg
delays, `services/historify_service.py` batch delays.

### 14. Timeouts

| Timeout | Value | Source | Confidence |
|---|---|---|---|
| Broker REST client (all requests) | 120 s | `utils/httpx_client.py#_create_http_client` | verified_code |
| httpx keep-alive expiry | 30 s | same | verified_code |
| SQLite busy timeout | 15000 ms | `database/__init__.py` | verified_code |
| Postgres-style pool timeout (non-SQLite) | 10 s (`pool_size=50`, `max_overflow=100`) | `database/engine_factory.py#create_db_engine` | verified_code |
| WS server ping interval / pong timeout | 20 s / 20 s | `WEBSOCKET_PROXY` config; `websocket_proxy/server.py#WebSocketProxy.start` | verified_code |
| WS unauthenticated grace | 15 s | `WS_AUTH_GRACE_SECONDS` | verified_code |
| Stale-tick warning threshold | 120 s | `WS_STALE_TICK_WARN_SECONDS` | verified_code |
| MCP OAuth access token | 900 s default, **hard cap 3600** | `.sample.env` (advanced block) | inferred_convention |
| MCP refresh / code TTL | 2592000 s / 60 s (documented defaults) | `.sample.env` | inferred_convention |
| Rate-limit window | moving-window, in-memory | `limiter.py#limiter` | verified_code |

### 15. Polling

| Poll | Interval | Source | Confidence |
|---|---|---|---|
| Sandbox execution engine tick | `order_check_interval` config, default 5 s | `sandbox/execution_thread.py#ExecutionEngineThread.__init__` | verified_code |
| Order-update REST polling (groww, fivepaisa, samco, flattrade) | `ORDER_POLL_INTERVAL=5 s` | `.sample.env`, `services/order_update_service.py#_POLLING_BROKERS` | verified_code |
| Broker connection keep-warm ping | `BROKER_KEEPALIVE_INTERVAL=20 s` (floor 5) | `services/broker_keepalive_service.py` | verified_code |
| Active-broker re-resolution in the keep-warm loop | 300 s | `services/broker_keepalive_service.py#_BROKER_REFRESH_SECONDS` | verified_code |
| Health metrics sampling | 10 s | `HEALTH_SAMPLE_INTERVAL` default in `utils/health_monitor.py` | verified_code |
| WS proxy stats-file write / stale check / cleanup | drive 0.5 s shutdown poll; stale check ≤ every 30 s; cleanup every 300 s | `websocket_proxy/server.py#WebSocketProxy` | verified_code |
| Market-data service health loop | `HEALTH_CHECK_INTERVAL`; a 300 s sleep is present in the service | `services/market_data_service.py` | verified_code |

### 16. Delays (explicit sleeps)

| Delay | Where | Purpose | Confidence |
|---|---|---|---|
| `time.sleep(1.0)` between basket batches | `services/basket_order_service.py` | inbound rate-limit compliance | verified_code |
| `order_delay` between split/options legs | `services/split_order_service.py`, `services/options_multiorder_service.py`, `services/place_options_order_service.py` | rate-limit compliance | verified_code |
| 1 s loop granularity | `sandbox/execution_thread.py#ExecutionEngineThread.run` | allows prompt shutdown while polling every N s | verified_code |
| `batch_delay` / `delay` in bulk paths | `services/historify_service.py`, `services/oi_profile_service.py` | chunk pacing, API quota | verified_code |
| `_MIN_HISTORY_INTERVAL - elapsed` | `services/history_service.py#_enforce_rate_limit` | minimum spacing between history calls | verified_code |
| `0.5 * attempt` | strategy-book boot retry | lock contention relief | verified_code |
| `time.sleep(sleep_seconds)` | `services/broker_keepalive_service.py` | loop cadence | verified_code |
| Short sleeps in bot/WS client loops (0.1–5 s) | `services/websocket_client.py`, `services/telegram_bot_service.py`, `services/whatsapp_bot_service.py` | reconnect and cooperativeness | verified_code |
| **Cron cadences** | | | |
| Sandbox MIS square-off NSE/BSE | `15:15` IST | `sandbox/squareoff_thread.py` | verified_code |
| Sandbox MIS square-off CDS/BCD | `16:45` IST | same | verified_code |
| Sandbox MIS square-off MCX | `23:30` IST | same | verified_code |
| Sandbox MIS square-off NCDEX | `17:00` IST | same | verified_code |
| Square-off backup sweep | every 1 minute | same | verified_code |
| T+1 settlement (CNC→holdings) | `00:00` IST | same | verified_code |
| Daily PnL snapshot | `23:59` IST | same | verified_code |
| Daily PnL reset | `SESSION_EXPIRY_TIME` (default `03:00` IST) | same | verified_code |
| Python-strategy daily maintenance | `00:01` IST | `blueprints/python_strategy.py` | verified_code |
| Strategy/user-defined schedule | user cron (hh:mm, days) | `blueprints/python_strategy.py`, `blueprints/strategy.py` | verified_code |
| Chartink schedule | user-defined | `blueprints/chartink.py` | verified_code |
| Flow jobs | interval (s/m/h) or cron | `services/flow_scheduler_service.py` | verified_code |
| Historify jobs | interval (h/m) or cron (Asia/Kolkata) | `services/historify_scheduler_service.py` | verified_code |

---

## Plane C — Cross-cutting Behaviour

### 17. Critical Paths

| Path | p99 latency | Latency target | Bottleneck operations | Confidence |
|---|---|---|---|---|
| FLOW-001 live order placement | unknown (no measurement found) | unknown | broker REST round trip; `get_auth_token_broker` cache miss; first-order module import | inferred_convention (structure) |
| FLOW-003 tick → browser | unknown | unknown | single ZMQ SUB listener; per-client asyncio send; 50 ms per-key throttle | inferred_convention |
| FLOW-005 sandbox fill cycle | unknown | unknown | one `get_multiquotes` batch per 5 s cycle | inferred_convention |
| FLOW-002 login → first feed | unknown | unknown | broker OAuth; lazy broker-module import (~3.5 s for all 30 auth modules, avoided by lazy loading) | verified_code for the lazy-load rationale |

**The keep-warm service exists precisely because the order path's first leg after >30 s idle pays a
TCP+TLS handshake (~100–150 ms, per its own docstring)** — the only latency figure quoted anywhere in
the sources inspected, and it is a documented observation, not a measured p99 for the whole path.

### 18. Error & Recovery Flows

| Scenario | Detection | Recovery | User-facing outcome | Source |
|---|---|---|---|---|
| Missing/short `APP_KEY` | process boot | refuse to start | `RuntimeError` text with a generate command | `app.py#create_app` |
| Debug on a public interface | `__main__` start | refuse to start | stderr banner + `exit(1)` | `app.py` |
| MCP + debug / MCP without public URL | boot | refuse to start | `RuntimeError` | `app.py#create_app` |
| CSRF failure (400) | errorhandler | JSON for `/api*`, flash+redirect for HTML | "Security token expired…" | `app.py#csrf_error` |
| Unknown route (404) | errorhandler | JSON for `/api/`, `/flow/api/`, `/flow/webhook/`; plain text for asset paths; else React shell | never 200-for-missing | `app.py#not_found_error` |
| Rate limit (429) | errorhandler | JSON for `/api/` with `retry_after: 60`; else redirect | "Rate limit exceeded…" | `app.py#rate_limit_exceeded` |
| Server error (500) | errorhandler | redirect to `/error` | React error page | `app.py#internal_server_error` |
| Broker module missing | import failure | publish `OrderFailedEvent`, return 404 | "Broker-specific module not found" | `services/place_order_service.py#place_order_with_auth` |
| Invalid API key on place | token lookup | return 403 **without logging** (anti-flood) | "Invalid openalgo apikey" | same |
| Broker call raises | exception in `place_order_api` | publish `OrderFailedEvent`, 500 | "Failed to place order due to internal error" (internals not leaked) | same |
| EventBus saturated | `_pending >= max_pending` | drop newest, sparse warning | none (side effects are best-effort) | `utils/event_bus.py#EventBus.publish` |
| Stale broker token after rollover | boot scan + `has_login_this_trading_session` | skip adapter start, log guidance | feed starts on next login | `services/order_update_service.py#start_order_update_adapters_on_boot` |
| Stale fill price | `quote_looks_stale` | defer fill | order remains pending | `sandbox/execution_engine.py` |
| DB init failure for one store | future result | log exception, continue the other 19 | boot proceeds | `app.py#setup_environment` |
| Strategy book unavailable at boot | 3 failed attempts | log error, continue boot | per-strategy P&L reports "error", never a false zero | `app.py#setup_environment` |
| WS port in use | `is_port_in_use` grace wait 2 s | raise with remediation steps | process refuses to serve ticks | `websocket_proxy/server.py#WebSocketProxy.__init__` |
| Tick implies a > 20% move or is > 60 s old | `MarketDataValidator.validate` | flag as a warning, do not silently accept | tick carries a validation warning | `services/market_data_service.py#MarketDataValidator` |
| Broker/DB write fails mid-operation | `except` blocks across `blueprints/*.py`, `broker/*/database/master_contract_db.py` | `session.rollback()` then surface the error | operation reported failed; no partial commit | `blueprints/admin.py`, `blueprints/sandbox.py`, `blueprints/security.py` |
| **Circuit breaker / compensation / DLQ** | — | **absent** — see the absences note in section 6b | — | searched, zero matches |

### 19. Configuration Behaviour

Configuration is a single flat `.env` read through `os.getenv` (with `utils/env_config.py#env_int` /
`#env_float` giving tolerant, bounded readers for numeric tuning so a typo cannot take the platform
down). Key groups (values as declared in `.sample.env`, which is the shipped template):

- **Secrets:** `APP_KEY`, `API_KEY_PEPPER`, `FERNET_SALT` — placeholders that `utils/env_check.py`
  auto-rotates to `secrets.token_hex(32)` on first run; the Fernet KDF falls back to a legacy static
  salt with a one-time stderr warning if `FERNET_SALT` is absent.
- **Datastores:** `DATABASE_URL`, `SANDBOX_DATABASE_URL`, `LOGS_DATABASE_URL`, `LATENCY_DATABASE_URL`,
  `HEALTH_DATABASE_URL`, `HISTORIFY_DATABASE_URL`.
- **Network:** `FLASK_HOST_IP`, `FLASK_PORT`, `WEBSOCKET_HOST`, `WEBSOCKET_PORT`, `WEBSOCKET_URL`,
  `ZMQ_HOST`, `ZMQ_PORT`, `HOST_SERVER`, `NGROK_ALLOW`, `TRUST_PROXY_HEADERS`.
- **Session/rollover:** `SESSION_EXPIRY_TIME` (`03:00`), `DISABLE_SESSION_EXPIRY`,
  `MASTER_CONTRACT_CUTOFF_TIME` (`08:00` IST), `CRYPTO_MASTER_CONTRACT_CUTOFF_TIME` (`00:00` UTC).
- **Rate limits:** `LOGIN_RATE_LIMIT_MIN` (`5 per minute`), `LOGIN_RATE_LIMIT_HOUR` (`25 per hour`),
  `RESET_RATE_LIMIT` (`15 per hour`), `API_RATE_LIMIT` (`50 per second`),
  `ORDER_RATE_LIMIT` (`10 per second`), `SMART_ORDER_RATE_LIMIT` (`10 per second`),
  `WEBHOOK_RATE_LIMIT` (`100 per minute`), `STRATEGY_RATE_LIMIT` (`200 per minute`).
- **Security posture:** `CORS_ENABLED` + `CORS_ALLOWED_*` + `CORS_MAX_AGE`, `CSP_ENABLED` +
  `CSP_REPORT_ONLY` + per-directive `CSP_*`, `CSRF_ENABLED` + `CSRF_TIME_LIMIT`,
  `SESSION_COOKIE_NAME`, `CSRF_COOKIE_NAME`, `FLASK_DEBUG_ALLOW_EXTERNAL`.
- **Order updates:** `ORDER_UPDATES_ENABLED` (`TRUE`), `ORDER_POLL_INTERVAL` (`5`).
- **Connection health:** `BROKER_CONNECTION_KEEPALIVE` (`TRUE`), `BROKER_KEEPALIVE_INTERVAL` (`20`),
  `BROKER_KEEPALIVE_WINDOW` (`09:00-23:30`).
- **WS limits:** `WS_AUTH_GRACE_SECONDS` (`15`), `WS_MAX_QUEUE` (`1024`), `WS_PING_INTERVAL` (`20`),
  `WS_PING_TIMEOUT` (`20`), `MAX_SYMBOLS_PER_WEBSOCKET` (`1000`), `MAX_WEBSOCKET_CONNECTIONS` (`3`),
  `ENABLE_CONNECTION_POOLING` (`true`).
- **Health thresholds:** `HEALTH_MEMORY_WARNING_THRESHOLD` (`3000`), `HEALTH_MEMORY_CRITICAL_THRESHOLD`
  (`5001`), plus FD/DB/WS/thread thresholds with platform-aware defaults in `utils/health_monitor.py`.
- **Logging:** `LOG_TO_FILE`, `LOG_LEVEL`, `LOG_DIR`, `LOG_FORMAT`, `LOG_RETENTION` (`14`),
  `TRAFFIC_LOG_RETENTION_DAYS` (`30`), `LOG_COLORS`, `FORCE_COLOR`.
- **Strategy logs:** `STRATEGY_LOG_MAX_FILES` (`10`), `STRATEGY_LOG_MAX_SIZE_MB` (`50`),
  `STRATEGY_LOG_RETENTION_DAYS` (`7`).
- **Deployment:** `APP_MODE` (`standalone` in Docker disables HTTP/2 and moves the WS proxy to a
  separate process), `ENV_CONFIG_VERSION` (`1.0.7`).

**Confidence:** `verified_code` for the keys and defaults (`.sample.env` read in full) and for the
code's own reads (`app.py`, `utils/httpx_client.py`, `websocket_proxy/*`, `services/*`).

### 20. Feature Flags

| Flag | Evaluated at | Enabled branch | Disabled branch | Confidence |
|---|---|---|---|---|
| `MCP_HTTP_ENABLED` | `app.py#create_app` | registers `/mcp`, `/oauth/*`, `/.well-known/*`; adds boot warnings when write-scope is on or DCR auto-approval is on; initialises the OAuth DB + signing key | neither namespace is registered | verified_code |
| `MCP_OAUTH_WRITE_SCOPE_ENABLED` | boot + OAuth discovery | AI clients can place/modify/cancel real orders | read-only MCP | verified_code (boot warning); inferred_convention (discovery) |
| `MCP_OAUTH_REQUIRE_APPROVAL` | boot | new DCR clients stay pending until admin approval | auto-approve | verified_code (boot warning) |
| `ENABLE_CONNECTION_POOLING` | WS proxy pool | multiple WS connections per broker (symbol headroom) | single connection per broker (legacy) | inferred_convention (.sample.env documents it; the read site was not opened) |
| `ORDER_UPDATES_ENABLED` | `services/order_update_service.py#_order_updates_enabled` | per-user order-feed adapters start | ingestion disabled entirely | verified_code |
| `BROKER_CONNECTION_KEEPALIVE` | `services/broker_keepalive_service.py` | keep-warm ping loop runs in the market window | no pings | verified_code |
| `DISABLE_SESSION_EXPIRY` | `utils/session.py#is_session_expiry_disabled` | sessions never auto-expire (crypto) | daily 03:00 rollover | verified_code |
| `CORS_ENABLED` | `cors.py#get_cors_config` | CORS config applied | no CORS headers | verified_code (read in `create_app`); the config body is unverified_gap |
| `CSP_ENABLED` / `CSP_REPORT_ONLY` | `csp.py#apply_csp_middleware` | CSP headers added (report-only vs enforcing) | no CSP middleware | inferred_convention |
| `CSRF_ENABLED` | `app.py#create_app` | `WTF_CSRF_ENABLED=True` | CSRF off | verified_code |
| `FLASK_DEBUG` | `__main__`, MCP guard | interactive debugger (loopback only) | normal serving | verified_code |
| `NGROK_ALLOW` | `__main__` | starts an ngrok tunnel | no tunnel | verified_code |
| `TRUST_PROXY_HEADERS` | IP helper | forwarded-IP headers trusted | headers ignored (anti-spoofing) | verified_code (documented in `.sample.env`; read site not opened) |

### 21. Hidden Business Logic

| Location | Implicit rule | Risk | Confidence |
|---|---|---|---|
| `database/__init__.py#_set_sqlite_pragmas` | A WAL + `busy_timeout` listener registered on the `Engine` class applies to **every** SQLite connection in the process, including all 35 broker `master_contract_db.py` modules — a behaviour nothing in those modules declares | medium | verified_code |
| `app.py#create_app` blueprint order | Middleware wraps in sequence: socketio → CSRF → limiter → CORS → CSP → socketio-error → security (IP ban) → traffic logging. Traffic logging is outermost, so it observes the final status including bans; presentation order is not the order of effect | medium | verified_code |
| `services/order_update_service.py#_POLLING_BROKERS` | Four brokers silently use REST polling instead of a push feed, and the reasons (single-WS-per-user eviction, no order stream) are only in comments — the fallback is invisible to a user seeing a slower update cadence | medium | verified_code |
| `utils/event_bus.py#EventBus.publish` | Events are **dropped, not queued**, past 1000 pending; a slow Telegram/WhatsApp subscriber can silently lose alerts | medium | verified_code |
| `utils/traffic_logger.py` | Traffic-logging middleware skips `/static/`, `/favicon.ico`, `/api/v1/latency/logs`, `/traffic/**` — the traffic monitor cannot see its own traffic, and latency-log reads are untracked | low | verified_code |
| `app.py#wait_for_db_ready` | Every non-static request blocks up to 30 s for background DB init — an availability coupling between request handling and a boot thread | medium | verified_code |
| `app.py#setup_environment` strategy-book registration | The strategy book is registered **before** `db_ready` is set, and the ordering is load-bearing: `order.placed` carries the only copy of the strategy tag, so an order placed earlier is permanently unattributable | high | verified_code |
| `database/sandbox_db.py#SandboxOrders.gtt_leg_id` | Uniqueness on `gtt_leg_id` is the crash-safety mechanism preventing a re-armed GTT from creating a second child order — an invariant expressed only as a column property | medium | verified_code |
| `websocket_proxy/server.py#WebSocketProxy.__init__` | The SUB socket **binds** while all publishers connect — a reversal of the usual PUB-binds convention, adopted to make the rendezvous port deterministic across processes | medium | verified_code |
| `services/historify_scheduler_service.py` / `services/flow_scheduler_service.py` | Job definitions are persisted (APScheduler job stores initialised in the boot DB list), so jobs survive restarts and are reconciled at boot rather than redefined fresh | low | inferred_convention |
| `services/scalping_risk_monitor_service.py` | The browser and server both evaluate the same trailing-stop algorithm (`evaluateTrail` mirrored in `evaluate_trail`); a divergence between the two implementations would silently disagree about the stop level | medium | verified_code (mirroring is documented in the function); the browser side is unverified_gap |

### 22. Documentation Contradictions (drift)

| ID | Documented claim | Actual implementation | Impact | Confidence |
|---|---|---|---|---|
| DC-001 | `kb/*.md` are starter templates that state "these kb/ notes haven't been filled in yet" and describe a project with no code | The repository is a large, fully implemented trading platform (2,440 tracked files, 35 broker plugins, a React frontend) | The project's own KB gives no orientation; an agent trusting it would misjudge the project as empty. Tier 6 docs lose to Tier 3 code (rule: *if code conflicts with documentation, the code wins*) | verified_code |
| DC-002 | `kb/gotchas.md#WebSocketProxy.__init__` and the note text say the ZMQ port is **5555** | `websocket_proxy/server.py` defaults to `os.getenv("ZMQ_PORT", "5555")`, but the shipped `.sample.env` sets `ZMQ_PORT='9555'`; Tier 2 config wins, so a standard install runs **9555** | Any operator or agent acting on the gotcha note would watch the wrong port when diagnosing a bind conflict | verified_code |
| DC-003 | Product/brand is **OpenAlgo** (`.sample.env` header "OpenAlgo Environment Configuration File", the startup banner "OpenAlgo v{version}", docs links to openalgo.in) | The repository folder and launcher are `qwantonomous` (`run_qwantonomous.bat`), and `pyproject.toml` names the package `openalgoUI` while depending on `openalgo==2.0.3` | Branding and package identity disagree three ways in one tree; anything keyed on the product name (logs, installers, support) will be inconsistent | verified_code |
| DC-004 | `.sample.env` declares the WebSocket endpoint as `WEBSOCKET_PORT='9999'` / `ws://127.0.0.1:9999` | The code's own default is 8765 (`app.py` banner reads `os.getenv("WEBSOCKET_PORT", 8765)`; `docker-compose.yaml` maps `${WEBSOCKET_PORT:-8765}`) | A user who copies `.sample.env` runs on 9999 while Docker and the SDK examples assume 8765: a documented port that the SDK clients and container manifests do not share | verified_code |
| DC-005 | `.sample.env` documents `ZMQ_HOST`/`ZMQ_PORT` "Do NOT change this to 0.0.0.0" with `127.0.0.1` | Consistent with the code (the SUB binds `tcp://{ZMQ_HOST}:{ZMQ_PORT}` and publishers connect) — **no drift**, recorded to show the check was run | none | verified_code |
| DC-006 | `kb/architecture.md#Components` documents component `path/to/module` entries with an entry point at `path/to/file.py:NN` — a placeholder that also uses line numbers, which the project's own KB rules forbid | No such modules exist; the KB conventions require searchable `path#symbol` references | The KB demonstrates a reference style its own conventions reject, so following it would plant brittle pointers | verified_code |
| DC-007 | `.github/workflows/ci.yml` runs Ruff lint and format with `continue-on-error: true` ("pre-existing warnings in broker modules") and `bandit`/`pip-audit` also `continue-on-error: true` | These gates cannot fail the build — CI is green regardless of lint or security findings | A green CI badge does not evidence clean lint or a clean dependency audit; a security finding would pass silently | verified_code |
| DC-008 | `docs/releases/version-2.0.2.1-released.md` exists and `pyproject.toml` declares version `2.0.2.1`, while `.sample.env` self-identifies as `ENV_CONFIG_VERSION='1.0.7'` and `utils/version.py#get_version` is the runtime source | Three independent version cursors (product, env-template, runtime) with no demonstrated synchronisation | Version-mismatch diagnostics and upgrade prompts key off different numbers | inferred_convention (the release note and version.py bodies were not opened) |

### 23. Evidence Index

| Finding | Evidence type | Provenance | Confidence |
|---|---|---|---|
| Runtime platform & deps | manifest | `pyproject.toml` | verified_code |
| App factory, middleware order, error handlers | code | `app.py#create_app` | verified_code |
| Boot orchestration, DB init, schedulers | code | `app.py#setup_environment` | verified_code |
| API surface | code | `restx_api/__init__.py` | verified_code |
| Order validation & placement | code | `services/place_order_service.py#validate_order_data`, `#place_order` | verified_code |
| Order routing gate | code | `services/order_router_service.py#should_route_to_pending`, `#IMMEDIATE_EXECUTION_OPERATIONS` | verified_code |
| Domain enums | code | `utils/constants.py` | verified_code |
| Event bus + backpressure | code | `utils/event_bus.py#EventBus.publish` | verified_code |
| Subscription map | code | `subscribers/__init__.py#register_all` | verified_code |
| SQLite pragmas | code | `database/__init__.py#_set_sqlite_pragmas` | verified_code |
| Pooling policy | code | `database/engine_factory.py#create_db_engine` | verified_code |
| Auth secrets/KDF/caches | code | `database/auth_db.py` | verified_code |
| Sandbox schema & constraints | code | `database/sandbox_db.py#SandboxOrders`, `#SandboxPositions` | verified_code |
| Fill guard | code | `sandbox/execution_engine.py#quote_looks_stale` | verified_code |
| Cron jobs | code | `sandbox/squareoff_thread.py#_schedule_square_off_jobs` | verified_code |
| Execution engine cadence | code | `sandbox/execution_thread.py#ExecutionEngineThread.run` | verified_code |
| GTT maintenance cadence | code | `sandbox/execution_thread.py#GTTMaintenanceThread` | verified_code |
| HTTP client timeouts/pool | code | `utils/httpx_client.py#_create_http_client` | verified_code |
| Broker keep-warm | code | `services/broker_keepalive_service.py` | verified_code |
| Order-update adapters & polling set | code | `services/order_update_service.py#_POLLING_BROKERS`, `#start_order_update_adapters_on_boot` | verified_code |
| Scalping risk engine | code | `services/scalping_risk_monitor_service.py#evaluate_trail` | verified_code |
| Session/rollover rules | code | `utils/session.py#is_session_valid`, `#revoke_user_tokens`, `#get_trading_session_date` | verified_code |
| IP-ban middleware | code | `utils/security_middleware.py#SecurityMiddleware` | verified_code |
| Traffic logger | code | `utils/traffic_logger.py#TrafficLoggerMiddleware` | verified_code |
| Rate limiter | code | `limiter.py#limiter`, `restx_api/place_order.py` | verified_code |
| WS proxy topology & limits | code | `websocket_proxy/server.py#WebSocketProxy.__init__`, `#start` | verified_code |
| WS pooling & ZMQ publisher | code | `websocket_proxy/connection_manager.py#SharedZmqPublisher`, `#get_max_symbols_per_websocket` | verified_code |
| Plugin contract | code + data | `utils/plugin_loader.py#load_broker_capabilities`, `broker/zerodha/plugin.json` | verified_code |
| Broker retry/backoff samples | code | `broker/angel/api/data.py#get_api_response`, `broker/dhan/api/data.py#get_api_response`, `broker/definedge/api/rate_limiter.py` | verified_code |
| Config surface | config | `.sample.env` | verified_code |
| CI reality (non-blocking gates) | CI manifest | `.github/workflows/ci.yml` | verified_code |
| Container topology | deploy manifest | `docker-compose.yaml` | verified_code |
| KT template state | doc | `kb/overview.md`, `kb/architecture.md`, `kb/gotchas.md` | verified_code |

### 24. Unknown / Unverified Behaviour

| ID | Subsystem | Unresolved question | How to verify |
|---|---|---|---|
| U-01 | Frontend WS manager | `kb/gotchas.md` cites `frontend/src/lib/MarketDataManager.ts#MarketDataManager` for the reconnect rule; the file was not opened in this pass | open the file and confirm the `wasClean` handling |
| U-02 | CORS/CSP bodies | `cors.py#get_cors_config` and `csp.py#apply_csp_middleware` are called at boot but their bodies were not read | read both modules and diff against the `CORS_*`/`CSP_*` keys |
| U-03 | Call graph completeness | Only a representative edge set was traced; no full symbol graph was built | generate a real call graph (static analysis) and compare |
| U-04 | Latency figures | No p99 or measured latency for any path; the only figure found is the keep-warm handshake estimate (~100–150 ms) | instrument the request path or read `latency.db` in a running install |
| U-05 | Telegram triple implementation | `services/telegram_bot_service.py`, `telegram_bot_service_fixed.py`, and `telegram_bot_service_v2.py` all exist with a `TelegramBotService` class; which is live, and whether the other two are dead code, was not established | grep for imports of each module |
| U-06 | MCP OAuth internals | Only the registration gate, the CSRF exemptions and the boot warnings were read; token/refresh/revocation flow bodies were not | read `blueprints/mcp_oauth.py`, `utils/oauth_keys.py`, `database/oauth_db.py` |
| U-07 | Per-broker timeouts | Timeouts/backoffs exist per broker plugin; only angel, dhan and definedge were sampled | sweep `broker/*/api/*.py` for timeout constants |
| U-08 | Historify download engine | Scheduler and pacing were read; the actual download/candle-write engine was not | read `services/historify_service.py` + `database/historify_db.py` |
| U-09 | Frontend build state | `is_react_frontend_available()` decides whether the SPA is served; whether `frontend/dist` exists in a given checkout is environment-dependent | check `frontend/dist/` |
| U-10 | Flow engine node semantics | Node contracts and the validator were mapped by signature; the per-node execution semantics of ~70 node types were not read | read `services/flow_executor_service.py` and `docs/prd/flow-node-reference.md` |
| U-11 | Upgrade/migration path | `upgrade/migrate_all.py` (the orchestrator and its 23-entry order) is now read and recorded; the body of each individual migration script is not | read each `upgrade/migrate_*.py` and dry-run against a copy of the DB |
| U-12 | Distributed tracing | Confirmed absent by search (zero matches for otel/jaeger/zipkin/tracer). No span data exists | n/a — confirmed absence, not a gap to close |
| U-13 | Tenancy scoping | Confirmed absent by search (zero matches for tenant). Ownership is per-`user_id`, not per-tenant | n/a — confirmed absence |
| U-14 | Circuit breaker / compensation / DLQ | Confirmed absent: the only "circuit breaker" is a price-spike constant in `MarketDataValidator`; the only compensation is per-operation `rollback()`; no saga, two-phase commit or dead-letter queue exists | n/a — confirmed absence. If these are wanted they are new features, not extraction gaps |
| U-15 | Non-SQLite backends | `database/engine_factory.py` branches to a `QueuePool` for PostgreSQL, but no Postgres URL, schema or migration exists in the repo, so that path is unexercised here | run against a Postgres `DATABASE_URL` to observe the pooled path |

---

## Verification Gate

Per the skill's gate, this domain's check was run against the real artefact rather than asserted:

1. **JSON Output-Contract validity** — `audit/behavioral-specification.json` parsed with Python's
   `json` module (and `json.tool`) to prove it is well-formed.
2. **Provenance resolution** — every `path#symbol` citation in both artefacts was resolved against
   the working tree: the file must exist and the symbol must appear in it (or the `:line` form must be
   in range). Any unresolved citation is reported, not hidden.
3. **No source changes** — this is a read-only extraction; no project code was modified, so the
   project's own `pytest` suite is not in scope for this change (per the targeted-testing rule).
   The command actually run and its result are recorded in the completion message, not here.

**Confidence statement:** every `verified_code` row above rests on a file this session opened; every
row that could not be so grounded is tagged `inferred_convention` or `unverified_gap` and listed in
section 24. No unverified claim is presented as fact.
