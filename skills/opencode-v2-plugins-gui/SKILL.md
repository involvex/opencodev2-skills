---
name: opencode-v2-plugins-gui
description: Build OpenCode Desktop GUI plugins with @opencode/plugin/gui. Use when adding panels, commands, settings pages, slots, IPC between main/window, or Electron/main-process integrations for the desktop app.
---

# OpenCode V2 GUI Plugins (Desktop)

Docs: https://opencode.ai/v2/docs/build/plugins/gui  
GUI plugins are experimental (`gui-experimental`). They can ship alongside server/TUI parts in one package.

## Package layout

```json title="package.json"
{
  "name": "opencode-acme-plugin",
  "type": "module",
  "exports": {
    ".": "./src/index.ts",           // server part (optional)
    "./tui": "./src/tui.tsx",        // TUI part (optional)
    "./gui-experimental": "./dist/gui-experimental.cjs"
  },
  "dependencies": { "@opencode/plugin": "latest" },
  "devDependencies": {
    "vite": "^7.0.0",
    "vite-plugin-solid": "^2.11.0",
    "solid-js": "^1.9.0",
    "electron": "^X.Y.Z",
    "@types/node": "^X.Y.Z"
  }
}
```

- GUI takes plugin id from server part id, else package name (directory name for local dir without name). Read as `ctx.id`.
- Users list in `opencode.jsonc` (desktop reads only `opencode.jsonc`, not `cli.json`).
- One bundle serves both processes; app loads from its own background server.

## Entry & two processes

```tsx title="src/gui.tsx"
import { shell } from "electron"
import { Plugin } from "@opencode/plugin/gui"

export default Plugin.define({
  setup(ctx) {
    // Window: Solid, ctx.layout, ctx.add(Command,...)
  },
  main(ctx) {
    // Main (Electron): Node/Electron, ctx.windows
    ctx.windows.on("open", () => shell.beep())
  },
})
```

- `setup` runs in every desktop window
- `main` runs once in Electron main process
- Both evaluate same file; placeholders for the other process' modules
- `Plugin.define` same in both; window-only exports (`Command`, `bindPlugin`, `createKeyed`) are `undefined` in main

## Build (Vite preset)

```ts title="vite.config.ts"
import { defineConfig } from "vite"
import { gui } from "@opencode/plugin/gui/vite"

export default defineConfig({ plugins: [gui()] })
```

```bash
vite build  # or vite build --watch for dev
```

Preset:
- Compiles Solid JSX via `vite-plugin-solid`
- Emits single CommonJS file, inlines dynamic `import()`
- Externalizes app shared modules, Electron, Node built-ins; bundles rest
- Fails on other `effect/*`, `solid-js/*`, `@opencode/*` not provided - import from shared entry

Options: `gui({ entry: "src/desktop.tsx", outFile: "dist/desktop.cjs" })` (defaults: `src/gui.tsx` -> `dist/gui-experimental.cjs`)

### Shared modules
- Both: `effect`, `@opencode/plugin/gui`, `@opencode/plugin/gui/main`, `@opencode/plugin/rpc`, `@opencode/schema/rpc`, `@opencode/client`, `@opencode/client/effect`
- Windows only: `solid-js`, `solid-js/web`, `solid-js/store`, `@opencode/client/solid`, `@opencode/plugin/gui/{file,review,browser}`
- Main only: `electron`, Node built-ins

## Definition

```ts
import { Schema } from "effect"
import { Plugin, Store } from "@opencode/plugin/gui"

export const definition = Plugin.define({
  os: ["macos", "linux"], // omit for all
  stores: {
    prefs: Store.global(Schema.Struct({ enabled: Schema.Boolean }), { enabled: true }),
  },
  i18n: {
    en: { title: "Acme", "command.open": "Open Acme" },
    de: { title: "Acme", "command.open": "Acme öffnen" },
  },
  setup(ctx) { /* window */ },
  main(ctx) { /* main */ },
})
```

- `ctx.id` prefixed on all ids created by plugin
- `os` restricts OS (plugin won't load if not matching)
- `stores` declared here, read as `ctx.stores.<key>`
- `i18n` inline; English fills missing keys. `ctx.t(key,params)`, `ctx.plural(key,count)`; in window also reads app `common.*`. In main missing returns key.
- Can split: define declarations separately, type setups with `Plugin.Setup<typeof definition>`, `Plugin.MainSetup`

## Context & APIs

Shared: `ctx.options`, `ctx.client` (server RPC), `ctx.app` (`{name, version, channel, platform, packaged}`), `ctx.signal`
Registries/tokens: `ctx.add`, `ctx.list`, `ctx.provide`, `ctx.use`, `ctx.main`
i18n: `ctx.t`, `ctx.plural`
Host: `ctx.layout`, `ctx.sessions`, `ctx.screen`, `ctx.servers`, `ctx.storage`, `ctx.dialogs`, `ctx.toast`, `ctx.attention`, `ctx.tabs`, `ctx.keybinds`, `ctx.commands`, `ctx.links`, `ctx.embeds`, `ctx.ui`, `ctx.system`, `ctx.desktop`, `ctx.locale`, `ctx.appearance`, `ctx.router`, `ctx.workspaces`

In components use `usePlugin<typeof definition>()`. Wrap provided components with `bindPlugin()` when passed to contracts.

## Contracts, IPC, Registries

### Contracts (window<->window)
```ts title="src/gui/contract.ts"
import type { JSX } from "solid-js"
import { Contract } from "@opencode/plugin/gui"

export interface AcmeBadge { Badge(props:{label:string}): JSX.Element; count(): number }
export const AcmeBadge = Contract.define<AcmeBadge, "acme.badge">("acme.badge")
```
Provide: `ctx.provide(AcmeBadge, { Badge: bindPlugin((p)=>...), count: ()=>42 })` (withdraw via returned cleanup).  
Consume: `const badge = ctx.use(AcmeBadge)` returns accessor `() => Live<T>`; read `live.status === "active" ? live.value... : fallback`.

### IPC (main<->window)
```ts
import { Schema } from "effect"
import { Ipc } from "@opencode/plugin/gui"

export const KeepAwake = Ipc.define({
  id: "acme.awake",
  methods: {
    screenActive: { output: Schema.Boolean },
    setScreenActive: { input: Schema.Boolean },
  },
})
```
Main provides: `ctx.provide(KeepAwake, impl)`. Window calls: `const awake = ctx.main(KeepAwake); await awake.setScreenActive(true)`. Others can `ctx.use(KeepAwake)`.

### Registries (plugin-owned lists)
```ts
export const Exporter = Registry.define<Exporter>("acme.exporter")
ctx.add(Exporter, () => ({ id:"gist", title: ctx.t("exporter.gist"), run:(t)=>upload(t) }))
const exporters = () => ctx.list(Exporter)
```

## Window contributions

- **Command**: palette/shortcuts/`/` menu. `run` can return `false` to pass key through. `bind`, `slash`, `enabled` reactive, `scope`/`editable`.
- **Panel** (`region:"side"`): tabs, can move to bottom pane. `list` returns tabs (stable objects), `render(props:{tab,session,screen})`, `focus/enter/close/closed/restore/normalize`. `mobile`, `wide`, `transient`. Reopen closed tabs via `closed`/`restore`.
- **SettingsPage**: in Settings, or section on host page. `entries` for search, `render({target})`.
- **Slot**: `window.bottom`, `session.header`, `session.location.menu`, `session.panel.end`. `order` controls sorting.
- **MenuItem**: in `session.panel`, `server.add`, `server.row`. Can be primary launcher.
- **TitlebarItem**: pill button with hover label, `busy`/`pressed`.
- **ComposerPill**: status pill above composer per session (menu or click). `show(input)` returns view or `undefined`.
- **LinkHandler**: handle local/custom links by `match`/`open`/`exists`, `priority`.
- **Server**: contribute server connections (multi-process aware). `ready`, entries with `http`, `reconnect/connect/remove`, optional `row`/`cover`.
- **Style**: inject CSS string.

## Main process (`main(ctx)`)

Runs once in Electron main. Use for:
- Native menu items via main registries (e.g. `MenubarItem` from `@opencode/plugin/gui/main`)
- `WebContentsView` embeds
- `Ipc` implementations with `ctx.provide(Ipc, impl)`
- Node/Electron APIs, windows management via `ctx.windows`
- Typed contracts exposed to windows

Remember: in main, window-only exports undefined; use main SDK from `@opencode/plugin/gui/main` for main-specific registries.

## Development tips

- `vite build --watch` reloads GUI in every window and main process (server entry not restarted). Changing options reloads plugin.
- Local dir with `name`+`exports` resolves `./gui-experimental`; else root `gui-experimental.cjs`/`.js`. Must be `.cjs` or `.js`.
- Plugin disabled in Settings > Extensions > Plugins; remove from `opencode.jsonc` or `opencode plugin remove`.
- If plugin id reserved/nested/under `opencode` namespace, GUI won't load.
- Keep module scope light; do expensive work in components/effects.
- For components passed across plugin boundaries, use `bindPlugin()` and annotate props.
- In panel `render`, don't destructure `props.session/tab` at top - read as getters (they update in place).
- Host APIs safe before mount (defaults/queued writes).

## Quick checklist

- [ ] Add `./gui-experimental` export pointing to built `.cjs`
- [ ] `vite.config.ts` uses `gui()` preset
- [ ] `src/gui.tsx` default-exports `Plugin.define({ setup?, main? })`
- [ ] Declare `os`, `stores`, `i18n` as needed
- [ ] Use shared modules only as allowed; bundle others
- [ ] Put tokens in `contract.ts`, use `Contract`/`Ipc`/`Registry`
- [ ] For cross-plugin components use `bindPlugin()`
- [ ] Test in desktop app with local dir + watch
