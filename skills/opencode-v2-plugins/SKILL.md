---
name: opencode-v2-plugins
description: Build or port OpenCode V2 plugins. Use when writing a new plugin, migrating a V1 plugin, or auto-starting companion processes like opencode serve.
---

# OpenCode V2 Plugins

Docs: https://opencode.ai/v2/docs/build/plugins and
https://opencode.ai/v2/docs/migrate-v1. V1 plugin implementations DO NOT run
in V2 — porting is mandatory, not optional.

## Packages

```json
{ "dependencies": { "@opencode/plugin": "^2.x", "@opencode/sdk": "^2.x" } }
```

`@opencode-ai/plugin` / `@opencode-ai/sdk` are V1. `@opencode/sdk` v2 exports
no `Config` type — use structural types instead.

## Entry shape

```ts
import { Plugin } from "@opencode/plugin";

export default Plugin.define({
  id: "my-plugin",
  setup(ctx) {
    const options = ctx.options; // plugin options object
    // ...hooks/tools registration...
  },
});
```

V2 has no `client.app.log` / `client.config.get` inside setup. Use process
logging and `ctx.storage` for persisted state.

## Config form

```jsonc
{ "plugins": ["some-package", { "package": "./local", "options": { "port": 4000 } }] }
```

## V1 -> V2 porting checklist

- [ ] New entrypoint (`Plugin.define`), new packages, `tsc --noEmit` clean
- [ ] Replace spawned `opencode web ...` with `opencode serve --hostname
      <h> --port <p> [--cors ...]` (verified flags on CLI 2.0.20)
- [ ] Drop reads of `server.*` from opencode.json (V2 ignores the key);
      precedence: plugin options > `OPENCODE_*` env > built-in defaults
- [ ] Drop mDNS support (`serve` has no mDNS flags)
- [ ] Liveness probes: `serve` process check + `/api/info` (not `/global/health`)
- [ ] Detached children intentionally outlive OpenCode — document it, and
      return a no-op cleanup unless supervision is wanted
