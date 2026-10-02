---
name: opencode-v2-mobile
description: Connect a mobile or remote client to OpenCode V2. Use when wiring phone apps, handling auth, scoping directories, or adapting to V2 session/message/form shapes.
---

# OpenCode V2 Mobile Integration

Companion skills: `opencode-v2-api` (routes), `opencode-v2-serve` (exposure).

## Connection

- `baseUrl = http://<lan-ip>:<port>`, no trailing slash (normalize it —
  double slashes break every request while probes still look fine).
- Basic auth on every request (`opencode` + user-set password, user-enterable
  per connection, never hardcoded). 401 = wrong/missing credentials.
- Connection test: authenticated `GET /api/info` must return a version.
  Health-probe success does NOT imply session/message calls work — verify a
  list call too.

## Sessions and directories

- Sessions are location-scoped (`location.directory`). Create with
  `location: {directory}` plus the directory header, or new sessions land in
  the server's CWD and later reads 404 from the wrong scope.
- IDs start with `ses` / `msg_`. `InvalidRequestError` on an ID means the
  client sent an empty or malformed one (often a missed `{data}` unwrap —
  every V2 response is enveloped).
- Sending is fire-and-forget `POST .../prompt {text, files[]}`; the reply
  arrives over `GET /api/event` SSE. If sends "go nowhere", check the
  server-side default model first: a 402/quota model fails the run with no
  client-visible error unless the app surfaces `finish: "error"`.

## Models, permissions, forms

- Per-session model: `POST .../model {model: {id, providerID}}` with BARE
  ids from `GET /api/model`. A usable default belongs in server config
  (`"model": "provider/model"`) so clients work with zero setup.
- Approvals are session-scoped (`/api/session/{id}/permission/...`); the
  sessionID is required — notification handlers must carry it.
- Questions are now forms (`/api/form`, reply `{answer: {key: value}}`).
  There is NO reject endpoint — surface that explicitly instead of failing
  silently.
- Missing V2 equivalents (file roots/status, server paths): use
  404-tolerant fallbacks, never crash the client on 404.
