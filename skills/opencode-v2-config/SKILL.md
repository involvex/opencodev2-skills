---
name: opencode-v2-config
description: Write native V2 opencode.json configuration. Use when creating or migrating server/project config, agents, permissions, providers, MCP servers, or plugins entries.
---

# OpenCode V2 Configuration

Docs: https://opencode.ai/v2/docs/config and https://opencode.ai/v2/docs/migrate-v1.
Format: JSON or JSONC with `"$schema": "https://opencode.ai/config.json"`.

Locations (merged farthest-ancestor-first, `.opencode/` variants override):

```text
~/.config/opencode/opencode.json(c)          global
<project>/opencode.json(c)                   project
<project>/.opencode/opencode.json(c)         project (higher precedence)
```

Terminal-only settings (theme, keybinds) belong in `~/.config/opencode/cli.json`,
never in `opencode.json`.

## V1 -> V2 renames (apply exactly these)

| V1 | V2 |
|---|---|
| `agent` map | `agents` map (`prompt`->`system`, `disable`->`disabled`, `permission`->`permissions[]`, drop `tools`/`options:{}`) |
| `mode` map | `agents` entries with `mode: "primary"` |
| `permission: {tool: effect}` | `permissions: [{action, resource, effect}]` ordered, last match wins |
| `bash` / `task` actions | `shell` / `subagent` (`write`/`patch` -> `edit`) |
| `bash(cmd)` keys | `{action:"shell", resource:"cmd *", effect}` (broad allows first, specific denies last) |
| `command` map | `commands` map (`subtask`->`subagent`) |
| `provider` map | `providers` (`npm`->`package`, `api`->`settings.baseURL`, `options`->`settings`) |
| `plugin` array | `plugins` array; `["pkg", opts]` tuple -> `{package, options}` |
| `autoupdate` bool | `update`: `"auto"` / `"notify"` / `"disabled"` |
| `small_model` | `agents.title.model` |
| `skills: {paths, urls}` | `skills: [...]` single array |
| flat `mcp: {name: {...}}` | `mcp: {servers: {...}}` (`enabled`->inverse `disabled`) |
| `experimental.mcp_timeout` | `mcp.timeout: {catalog, execution}` |
| `compaction.preserve_recent_tokens` | `compaction.keep.tokens` (drop `prune`, `tail_turns` — no equivalent) |
| `snapshot` / `attachment` / `reference` | `snapshots` / `media` / `references` |
| `"model": "provider/model"` | top-level default model (same syntax, recommended to set) |

## Drop (intentionally unsupported, warn if kept)

`server`, `layout`, `theme` (in opencode.json), non-hex agent `color`,
`continue_loop_on_deny`, V1 `web`/`tui` concepts. `disabled_providers` and
`enabled_providers` have no native field — keep in V1 syntax (accepted
without warning) or express via `experimental.policies`.

## Model switching contract

`POST /api/session/{id}/model` takes `{model: {id, providerID}}` with BARE
ids (`space-bunny-free`, not `opencode/space-bunny-free`). Get valid pairs
from `GET /api/model` (`id` + `providerID` fields).
