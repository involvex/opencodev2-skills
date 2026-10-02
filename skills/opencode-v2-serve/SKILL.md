---
name: opencode-v2-serve
description: Expose an OpenCode V2 server on the network. Use when setting up remote access, phone apps, pairing, passwords, CORS, or choosing between serve and the background service.
---

# OpenCode V2 Server Exposure

Docs: https://opencode.ai/v2/docs/cli and https://opencode.ai/v2/docs/troubleshooting.

## One server only

Sessions, plugins, and permissions live per server process. Never run the
background service AND `opencode serve` on different ports for the same
clients — state splits and each sees a different world.

| | Background service | `opencode serve` |
|---|---|---|
| Start | automatic (`opencode service start/stop/restart/status`) | foreground: `opencode serve --hostname 0.0.0.0 --port 4000` |
| Bind/persist | `opencode service set hostname 0.0.0.0` + `opencode service set port 4000` | flags per invocation |
| Best for | permanent host, phone apps, myfritz/tunnel routing | ad-hoc sharing, scripts |

`opencode web` does NOT exist in V2. Anything spawning it is a V1 leftover.

## Auth

- Basic auth, username `opencode` (override: `OPENCODE_SERVER_USERNAME`).
- Password: `OPENCODE_SERVER_PASSWORD` env, or service-managed
  (`opencode service get/set password`). A `serve` run without a set
  password prints an auto-generated one and rotates it every restart —
  always set a stable password for remote clients.
- Pair browser/web clients with `opencode pair --url http://<lan-ip>:<port>`.

## Remote checklist

1. Bind `0.0.0.0`, fixed port, stable password. Verify:
   `curl -u opencode:<pw> http://127.0.0.1:<port>/api/info` returns version.
2. Phone/LAN: `http://<lan-ip>:<port>` + same credentials. No CORS needed
   for native apps; browser origins need `opencode service set cors
   "http://origin:port"` (exact origin, comma-separated for several).
3. No mDNS in V2 (`serve` has no mDNS flags) — clients must use direct IPs.
4. Myfritz/reverse-proxy: plain HTTP works; keep the same host port stable.
5. If a client gets HTML instead of JSON, it is calling V1 routes — see the
   `opencode-v2-api` skill, not a server problem.

## PWA installation (Add to Home Screen)

The web UI is an installable PWA, but browsers only offer installation in a
**secure context** (plus the manifest + service worker the server ships):

- `http://127.0.0.1:<port>` / `http://localhost:<port>` → loopback is always
  secure → installable.
- `http://<lan-ip>:<port>` (e.g. `http://192.168.178.69:4000`) → plain HTTP
  on LAN is NOT secure → no install prompt ("cannot be installed"). Same
  app, same server version — only the origin differs.

Ways to get an installable origin for the PC server:

1. **HTTPS public URL (easiest)** — open the myfritz / tunnel / Tailscale
   `https://` address in the mobile browser instead of the LAN IP.
2. **Browser flag (LAN-only, no setup)** — in a Chromium browser open
   `chrome://flags`, set *Insecure origins treated as secure* to
   `http://<lan-ip>:<port>`, restart. Fragile across updates.
3. **HTTPS on the PC (robust LAN)** — Caddy/Nginx reverse proxy to
   `127.0.0.1:<port>` with a LAN-trusted cert (self-signed CA installed on
   Android, or Tailscale Serve / Cloudflare Tunnel for a real cert).
