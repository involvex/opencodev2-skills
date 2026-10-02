---
name: opencode-v2-api
description: Talk to the OpenCode V2 HTTP API. Use when calling server endpoints, migrating V1 clients, or debugging API responses. Covers /api/* routes, {data} envelopes, pagination, SSE events, and auth.
---

# OpenCode V2 API

Base: server root (e.g. `http://<host>:4000`). Every V2 route lives under
`/api/*`. Full reference: `GET /api` docs at https://opencode.ai/v2/docs/api
and machine-readable `https://opencode.ai/v2/openapi.json`.

## Auth

HTTP Basic. Username defaults to `opencode` (override with
`OPENCODE_SERVER_USERNAME`), password from `OPENCODE_SERVER_PASSWORD` or the
service config (`opencode service get password`). No-auth requests get
`401 {"_tag":"UnauthorizedError"}`.

## Envelopes and status codes

- Success bodies are wrapped: `{"data": ...}` (single) or
  `{"data": [...], "cursor": {"next": "..."|null}}` (paginated lists).
- Mutating endpoints often return `204 No Content` (PATCH, DELETE, replies,
  switch-model/agent). Empty body is success — do not call `.json()` on it.
- Unknown paths return the web UI `text/html` SPA fallback with 200 — if a
  client gets HTML where it expects JSON, it is calling a V1/wrong route.

## Key routes

```text
GET    /api/info                        server info + version
GET    /api/event                       SSE event stream
GET    /api/session                     list (global, ?parentID=null roots, ?limit ?cursor ?order)
POST   /api/session                     create {title?, agent?, model?, location?} -> {data}
GET    /api/session/{id}                get (includes .model, .location.directory)
PATCH  /api/session/{id}                {title?, metadata?} -> 204, then re-GET
DELETE /api/session/{id}                -> 204
GET    /api/session/{id}/message        -> {data: [...]} (user/assistant/idle/synthetic)
POST   /api/session/{id}/prompt         {text, files[]} fire-and-forget; reply arrives via SSE
POST   /api/session/{id}/command        {name, text, files} -> 204
POST   /api/session/{id}/interrupt      abort a running turn
GET    /api/session/{id}/diff           ?from=&to=
POST   /api/session/{id}/revert/stage   {messageID, partID?} + /revert/commit, DELETE /revert
POST   /api/session/{id}/model          {model: {id, providerID, variant?}} (bare ids, NOT provider/model)
POST   /api/session/{id}/agent          {agent}
GET    /api/permission/request          global pending approvals
POST   /api/session/{id}/permission/{req}/reply  {decision} -> 204
GET    /api/form                        pending forms (V2 replaced questions)
POST   /api/session/{id}/form/{form}/reply       {answer: {key: value}} -> 204
GET    /api/agent  /api/command  /api/provider  /api/model  /api/model/default  /api/config
GET    /api/fs/list  /api/fs/read/*  /api/fs/find     location-scoped files
GET    /api/pty ...                     PTY sessions (list/create/connect-token/...)
```

## V1 -> V2 route map (verified against CLI 2.0.20)

| V1 | V2 |
|---|---|
| `GET /global/health` | `GET /api/info` |
| `GET /global/event` | `GET /api/event` |
| `GET /experimental/session`, `GET /session` | `GET /api/session` |
| `POST /session` | `POST /api/session` |
| `GET/PATCH/DELETE /session/{id}` | same under `/api/session/{id}` |
| `GET /session/{id}/message` | `GET /api/session/{id}/message` |
| `POST /session/{id}/prompt_async {parts[], model?, agent?}` | switch model/agent first, then `POST .../prompt {text, files[]}` |
| `POST /session/{id}/abort` | `POST .../interrupt` |
| `POST /session/{id}/revert`, `/unrevert` | revert stage/commit flow (no unrevert) |
| `GET /permission`, `POST /permission/{id}/reply` | session-scoped permission routes |
| `/question/*` | forms API (no reject endpoint exists) |
| `/file`, `/file/roots`, `/file/status`, `/path` | `/api/fs/*` (no roots/status/path equivalents — use tolerant fallbacks) |

## Sessions are location-scoped

Session payloads carry `location.directory`, not a top-level `directory`.
Send the client's directory per request (header/body `location`) or sessions
land in the server's CWD. Session IDs start with `ses`; anything else in
`{sessionID}` yields `InvalidRequestError`.
