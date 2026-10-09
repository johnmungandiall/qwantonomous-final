# Broker credential endpoints — where broker configuration lives in this repo.

## The two things people confuse
- `/api/broker/credentials` (GET/POST) — `blueprints/broker_credentials.py`, blueprint prefix
  `/api/broker`. Session-cookie auth (`@check_session_validity`). Reads masked values from
  `os.getenv`, and on save **rewrites `.env`** via `#update_env_value`, answering
  `restart_required: True`. Consumed by this repo's own React page `frontend/src/pages/Profile.tsx`.
- `/api/v1/brokercredentials` (+ `/login`) — **DOES NOT EXIST here.** Verified project-wide: no such
  file in `restx_api/`, and zero `relogin` / `auto_login` matches anywhere in the Python sources.

## Why it matters
The Qwantonomous-Engine app (`C:\dev\Qwantonomous-Engine`) calls `/api/v1/brokercredentials` and
`/api/v1/brokercredentials/login` with an **API key** to store broker *login* credentials (client id,
password, TOTP secret) in an encrypted store, for headless auto-login after the nightly token revoke.
That is a different mechanism from this repo's `/api/broker/credentials`, which writes `.env` and
needs a restart. Do not conflate them — one is a dashboard feature of this repo, the other is a
contract this repo does not yet satisfy.

## Restart semantics
`POST /api/broker/credentials` always answers `restart_required: True`: the values are read with
`os.getenv` at request time and `blueprints/brlogin.py` also reads `BROKER_API_KEY` at module import,
so a change only takes effect after a process restart. Removing that restart is the point of the
q-server plan.

See [[gotchas]]; the engine-side contract is in long-term memory as `engine-q-server-contract`.
