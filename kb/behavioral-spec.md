# Behavioural Specification — the 24-dimension extraction of this platform.

Read-only extraction produced with `kbcodedev/business-logic-workflow-extractor` v2.1.0.
It answers "how does this system actually behave?" in one place, so nothing here needs
re-deriving. **It is a report, not code** — read it, do not cite it as the implementation.

## Artefacts
- `audit/BEHAVIORAL_SPECIFICATION.md` — the human report (24 sections, provenance + confidence per row).
- `audit/behavioral-specification.json` — the skill's machine-readable Output Contract (21 dimensions,
  same findings). Verified: parses, and every `path#symbol` citation resolves in the tree.

## How to use it
- Orientation: read the Summary table, then the plane that matches your question
  (Plane A rules/ingress/flows/state machines, Plane B storage/cache/queues/concurrency,
  Plane C config/drift).
- Confidence tags are load-bearing: `verified_code` was read directly; `inferred_convention`
  was reasoned; `unverified_gap` is listed in section 24 with a way to verify each.

## Drift it recorded (act on these, not on the stale sources)
- The `kb/` notes themselves are still the unfilled starter templates — see [[overview]] and
  [[architecture]]. The KB claims the project is empty; it is not.
- ZMQ port: `kb/gotchas.md` says 5555 but `.sample.env` sets `ZMQ_PORT='9555'` while
  `websocket_proxy/server.py#WebSocketProxy.__init__` defaults to 5555. Config wins — a standard
  install runs 9555.
- WebSocket port: `.sample.env` says 9999, while `app.py` (banner) and `docker-compose.yaml`
  both use 8765.
- Product identity disagrees three ways: folder/launcher `qwantonomous`, package `openalgoUI`,
  dependency `openalgo==2.0.3`.
- CI's Ruff, bandit and pip-audit steps all carry `continue-on-error: true` in
  `.github/workflows/ci.yml`, so a green build does not mean clean lint or a clean audit.

Cross-link: [[gotchas]] for the operational traps, [[architecture]] for the component map.
