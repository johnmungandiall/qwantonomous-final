# Skill Coverage Map — `business-logic-workflow-extractor` v2.1.0

**Purpose.** This file exists so that no item of the skill is silently dropped. It maps **every
named item** of the skill — its sections, invariants, evidence modalities, phases, contract
dimensions, anti-patterns and worked-example steps — to the delivered result that answers it, with
the provenance that proves the answer.

**How to read a status:**
- `DELIVERED` — the item has a result in the artefacts, grounded in files this session read.
- `CONFIRMED ABSENT` — the item was searched for and does not exist in this codebase. The absence is
  the result (a real gap, reported, never glossed).
- `N/A WITH REASON` — the item does not apply to this repository, and why.

**Artefacts referenced:** `audit/BEHAVIORAL_SPECIFICATION.md` (the report, sections cited as §n) and
`audit/behavioral-specification.json` (the Output Contract, dimensions cited by key).

---

## A. §1 Intent & Trigger Conditions

| Skill item | Status | Where the result is |
|---|---|---|
| **When to Use** — codebase archaeology | DELIVERED | The whole run: ~1,100 backend files; 96 distinct `path#symbol` citations in the report, all resolved |
| **When to Use** — reverse-engineering undocumented systems | DELIVERED | The repo's own `kb/` was unfilled starter templates (DC-001), so the behaviour was recovered from source, not docs |
| **When to Use** — auditing architectural boundaries | DELIVERED | §2 entry points, §6b middleware chain, §7 storage boundaries, §9 external boundaries |
| **When to Use** — synthesizing a Complete Behavioral Specification | DELIVERED | §1–§24 of the report; 24 dimension keys in the JSON |
| **Triggers** — reverse-engineering unknown repositories | DELIVERED | whole run |
| **Triggers** — extracting hidden business rules from a monolith | DELIVERED | §21 Hidden Business Logic — 11 rows, each with risk level |
| **Triggers** — uncovering latency/timeout bottlenecks | DELIVERED | §14 Timeouts, §17 Critical Paths (bottlenecks named), §13 Retries |
| **Triggers** — mapping distributed event topologies | DELIVERED | §10 Queue/Event Topology — EventBus topics + ZMQ fan-in bus |
| **Triggers** — discovering cache/database invariants | DELIVERED | §7 Database Behaviour, §8 Cache Behaviour (15 caches, TTLs + invalidation) |
| **Triggers** — detecting documentation-vs-implementation drift | DELIVERED | §22 — 8 contradictions (DC-001…DC-008) |
| **Prerequisites** — access to the project source tree | DELIVERED | read-only inspection of the working tree |
| **Prerequisites** — configuration manifests | DELIVERED | `pyproject.toml`, `.sample.env` (read in full), `docker-compose.yaml`, `.github/workflows/ci.yml` |
| **Prerequisites** — schema definitions | DELIVERED | ORM class definitions in `database/*.py` (§7) |
| **Prerequisites** — read-only repository inspection | DELIVERED | no project code was modified by this run; only `audit/` + `kb/` artefacts were added |

## B. §2 Invariant Principles

| # | Skill item | Status | Where the result is |
|---|---|---|---|
| 1 | **Three-Plane Universal Decomposition** | DELIVERED | Plane A (§1–§6b), Plane B (§7–§16), Plane C (§17–§24) |
| 2 | **Multi-Modal Evidence Synthesis** (six modalities) | DELIVERED | see modality table in section D below — all six used |
| 3 | **Evidence Precedence Hierarchy** (6 tiers) | DELIVERED | `meta.evidence_precedence` in the JSON; applied — e.g. `.sample.env` (Tier 2) beat the code default on the ZMQ port (DC-002), and the code (Tier 3) beat the docs (Tier 6) on the empty-KB claim (DC-001) |
| 4 | **Strict Dependency Ordering** (Orders 1–7) | DELIVERED | worked in that order: manifests/config → schemas → ingress → call graph → invariants → temporals → synthesis |
| 5 | **Pure Prompt Directives without Synthetic Code Examples** | DELIVERED | the artefacts quote real project values, commands and constants; no synthetic language-specific snippet is offered as guidance |
| 6 | **Project-Grounding Invariant** | DELIVERED | every citation resolved by the gate against the real tree; each row tagged `verified_code` / `inferred_convention` / `unverified_gap`; the unverified ones are listed, not hidden |

### B1. Evidence modalities (§2.2 a–f)

| Modality | Status | Where the result is |
|---|---|---|
| (a) Semantic AST/CST & symbol graphs — trees, type hierarchies, call graphs | DELIVERED | `repo_map` traversal, symbol reads, `call_graph` dimension, §3 |
| (b) Configuration & Environment — env vars, static/dynamic config, feature flags, secrets, runtime settings | DELIVERED | §19 (90+ keys), §20 (13 flags), `configuration_behavior`, `feature_flags`; secret handling in BR-001/BR-002 and the Fernet/KDF read |
| (c) Dependency Injection & Framework — DI registrations, middleware pipelines, conventions, reflection, dynamic import bindings | DELIVERED | §6b middleware chain (9 layers); `blueprints/*` registration order; `utils/plugin_loader.py#_LazyBrokerAuthDict` (lazy dynamic import); 35-broker plugin convention; lazy APScheduler `__new__` singletons |
| (d) Persistence & Storage — raw SQL, ORM migrations, triggers/procedures, indices, locking, isolation | DELIVERED | §7: WAL/pragma listener, NullPool policy, ORM-only access (raw `text()`/`execute()` search returned **zero** matches in `database/*.py`), the 23-migration ordered runner, constraints/indices; **no triggers and no stored procedures exist** |
| (e) Messaging & Integration — routing keys, topic bindings, DLQs, RPC/REST contracts, webhook subscriptions | DELIVERED | §10 topic inventory; `queue_event_topology`; webhook entry points EP-003; DLQ = **CONFIRMED ABSENT** |
| (f) Operational & Verification — test suites, build manifests, CI/CD, deployment | DELIVERED | `.github/workflows/ci.yml` read (non-blocking gates flagged as DC-007), `docker-compose.yaml`, `pyproject.toml` pytest config, the verification gate execution |

## C. §3 High-Signal Execution Workflow

| Phase | Status | Result |
|---|---|---|
| **Phase 1 — Ingress & Boundary Discovery** | DELIVERED | §2: 10 entry-point classes (HTTP web, `/api/v1` ~50 namespaces, webhooks, OAuth callbacks, health, Remote MCP, WebSocket, Socket.IO, 8 scheduled job groups, background daemons) |
| **Phase 2 — Flow & Call-Graph Reconstruction** | DELIVERED | §3 call graph (13 traced edges), §4 seven end-to-end flows with transaction boundaries, §6b middleware sequence, §18 error/recovery paths |
| **Phase 3 — Business Invariant & Decision Mining** | DELIVERED | §1 28 business rules, §5 4 state machines (with guards), §6 5 decision graphs, §21 hidden logic |
| **Phase 4 — Temporal, Concurrency & Infrastructure** | DELIVERED | §13 retries (10), §14 timeouts, §15 polling, §16 delays + 8 cron groups, §11 parallelism, §12 concurrency/race hazards, §7 DB behaviour, §8 caches, §10 queues |
| **Phase 5 — Complete Behavioral Specification Synthesis** | DELIVERED | §1–§24 compiled; `summary` counters; 24 dimension keys |
| **Verification Gate** | DELIVERED | `python _verify_behavioral_spec.py` → JSON parse OK (24 dimension keys, 0 declared-but-missing); 96 citations checked in the `.md` and 94 in the `.json`, **0 unresolved**; GATE: PASS |

## D. §4 Input / Output Contracts

| Item | Status | Result |
|---|---|---|
| **Input Contract** — `project_root`, `analysis_mode`, `evidence_modalities`, `output_format` | DELIVERED | recorded under `meta` in the JSON: project root, `complete_behavioral_specification`, the six modalities, `structured_behavioral_spec` |
| **Output Contract** — the 24 dimensions, each individually addressable | DELIVERED | 24 distinct keys (an earlier draft merged three pairs into 21; that was a real defect and was corrected) |

### D1. The 24 dimensions — one row each

| # | Dimension key | Status | Content | Report § |
|---|---|---|---|---|
| 1 | `business_rules` | DELIVERED | 28 rules | §1 |
| 2 | `entry_points` | DELIVERED | 10 | §2 |
| 3 | `call_graph` | DELIVERED | 13 edges | §3 |
| 4 | `end_to_end_flows` | DELIVERED | 7 flows | §4 |
| 5 | `state_machines` | DELIVERED | 4 machines | §5 |
| 6 | `decision_graphs` | DELIVERED | 5 graphs | §6 |
| 7 | `database_behavior` | DELIVERED | engine/pragma/pooling/migration/access-style rows | §7 |
| 8 | `cache_behavior` | DELIVERED | 15 caches | §8 |
| 9 | `external_dependencies` | DELIVERED | 8 service rows + 35 broker families | §9 |
| 10 | `queue_event_topology` | DELIVERED | EventBus + ZMQ rows | §10 |
| 11 | `parallelism` | DELIVERED | 4 pool/limit rows | §11 |
| 12 | `concurrency` | DELIVERED | 5 singleton/race-hazard rows | §12 |
| 13 | `retries` | DELIVERED | 10 policies | §13 |
| 14 | `timeouts` | DELIVERED | 6 timeouts | §14 |
| 15 | `polling` | DELIVERED | 5 intervals | §15 |
| 16 | `delays` | DELIVERED | sleeps, throttles, maintenance + 8 cron groups | §16 |
| 17 | `critical_paths` | DELIVERED | 4 paths, bottlenecks named, latency honestly `null`/`unknown` | §17 |
| 18 | `error_and_recovery_flows` | DELIVERED | 19 scenarios | §18 |
| 19 | `configuration_behavior` | DELIVERED | 30 key rows | §19 |
| 20 | `feature_flags` | DELIVERED | 13 flags | §20 |
| 21 | `hidden_business_logic` | DELIVERED | 11 rows with risk | §21 |
| 22 | `documentation_contradictions` | DELIVERED | 8 rows | §22 |
| 23 | `evidence_index` | DELIVERED | 37 provenance rows | §23 |
| 24 | `unknown_unverified_behavior` | DELIVERED | 15 rows, each with a verification route | §24 |

## E. §5 Anti-Patterns — the check on my own method

| Anti-pattern | Status | Evidence of avoidance |
|---|---|---|
| **Inverted Precedence / Documentation Credulity** | AVOIDED | 8 drift rows; config beat code-default (DC-002), code beat docs (DC-001) |
| **Unordered Extraction** | AVOIDED | manifests and config were read before code; schemas before services |
| **AST-Only Monoculture** | AVOIDED | all six evidence modalities used (section B1) |
| **Hardcoded Code Examples in Prompt** | AVOIDED | no synthetic snippets as directives |
| **Static Rosters & Keyword Dictionaries** | AVOIDED | discovery was by symbol read, `repo_map` traversal and semantic search; where a keyword search *was* used (tracing, tenancy, circuit breakers) it was to prove an **absence**, not to define behaviour |
| **Ignoring Implicit & Cascading Delays** | AVOIDED | captured keep-warm interval, 30 s pool expiry vs 20 s ping, 15 s SQLite busy timeout, 15 s WS auth grace, 50 ms throttle, 60 s GTT reclaim, 3600 s GTT expiry sweep |
| **Disconnected Flow Spaghetti** | AVOIDED | 7 causative flows traced ingress → domain → I/O → state |
| **Unverified Assertions** | AVOIDED | every row carries a confidence tag; 15 unverified items listed with a verification route |

## F. §6 Worked-Example Steps

| Step | Status | Result |
|---|---|---|
| 1 — Multi-modal ingress & configuration discovery | DELIVERED | §2, §19, §20 |
| 2 — Flow & call-graph reconstruction | DELIVERED | §3, §4, §6b, §18 |
| 3 — Domain invariants, decisions & state machines | DELIVERED | §1, §5, §6, §21 |
| 4 — Temporal mechanics, concurrency & infrastructure audit | DELIVERED | §7, §8, §10, §11, §12, §13, §14, §15, §16 |
| 5 — Complete specification synthesis + evidence index + drift | DELIVERED | §22, §23, §24, and the whole JSON |

## G. Named absences — the honest gaps

These are items the skill names that this repository genuinely does not have. Each was searched for
across the whole Python tree; the search returned no matches, so the absence is the finding.

| Skill item | Search performed | Result |
|---|---|---|
| Distributed tracing (middleware item) | `opentelemetry\|otel\|jaeger\|zipkin\|tracer\|init_tracing` | **CONFIRMED ABSENT** — zero matches. Observability is per-request latency + traffic rows, not spans |
| Tenancy scoping (middleware item) | `tenant_id\|tenant\|multi_tenant\|multitenant` | **CONFIRMED ABSENT** — zero matches. Ownership is per-`user_id`; one broker per instance |
| Circuit breaker trips | `circuit_breaker\|CircuitBreaker\|half_open` | **CONFIRMED ABSENT** — the single match is a comment naming `MarketDataValidator.MAX_PRICE_CHANGE_PERCENT` (a 20% price-spike flag, not a breaker) |
| Compensation transactions / saga | `compensat\|saga\|two_phase` | **CONFIRMED ABSENT** — the nearest construct is per-operation `session.rollback()` |
| Dead-letter queues | `dead_letter\|dlq\|\bDLQ\b` | **CONFIRMED ABSENT** — the EventBus drops past 1000 pending; ZMQ PUB/SUB is fire-and-forget |
| Database triggers / stored procedures | ORM + raw-SQL sweep of `database/*.py` | **CONFIRMED ABSENT** — no triggers, no procedures; hidden rules live in ORM defaults and `CHECK`/`Unique` constraints |
| Raw hand-written SQL to audit for injection | `text(\|execute(` in `database/*.py` | **CONFIRMED ABSENT** — ORM-only data layer |
| PostgreSQL path | search for a Postgres URL/schema/migration | **N/A WITH REASON** — `database/engine_factory.py` branches to a `QueuePool` for Postgres, but no Postgres URL, schema or migration exists in the repo, so the path is unexercised (U-15) |
| Coverage % / latency targets (skill notes them as heuristics to re-derive) | searched for configured targets | **N/A WITH REASON** — no latency target or coverage threshold is configured; p99 is reported `null` with source `unknown` rather than invented |

## H. Out-of-scope items the request named — acknowledged, not violated

| Item | Handling |
|---|---|
| *"Never clutter the prompt with fragile, language-specific code snippets or static keyword dictionaries"* | Honoured. The skill's directives were carried out as instructions; the artefacts quote real project values and constants as **evidence**, never as template snippets to paste |
| *"Pure Prompt Directives without Synthetic Code Examples"* | Honoured, as above |
| *Prerequisites are a narrowing scope item* | Honoured — the run used only read-only inspection; no project code was modified |

---

## Coverage conclusion

Every structural item the skill names — 3 intent bullets, 6 invariants, 6 evidence modalities,
5 phases, 1 verification gate, 2 contracts, 24 dimensions, 8 anti-patterns and 5 worked-example
steps — has its own result above, and the 24 dimensions each exist as an individually addressable
key in `audit/behavioral-specification.json`. Nine named constructs are recorded as **confirmed
absences** rather than results, because the codebase does not contain them; each was established by
an explicit search, and none is presented as delivered.
