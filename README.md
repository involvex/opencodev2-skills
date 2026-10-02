# OpenCode V2 Skills

Agent skills for working with OpenCode V2 — API, config, server exposure,
plugins, and mobile integration. Written against CLI 2.0.20 and the live
[V2 docs](https://opencode.ai/v2/docs/).

## Install

```sh
bunx skills add involvex/opencodev2-skills -g
```

## Skills

| Skill | When to use |
|---|---|
| `opencode-v2-api` | Calling `/api/*` endpoints, migrating V1 clients, debugging responses |
| `opencode-v2-config` | Writing or migrating native V2 `opencode.json` |
| `opencode-v2-serve` | Exposing a server (serve vs service, auth, pairing, remote access) |
| `opencode-v2-plugins` | Building or porting V2 plugins |
| `opencode-v2-mobile` | Wiring phone/remote clients (auth, directories, forms, models) |
