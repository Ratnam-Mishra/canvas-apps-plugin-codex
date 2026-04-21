# AGENTS.md — Canvas Apps Plugin (Codex)

This project is a Codex-adapted port of Microsoft's `power-platform-skills` canvas-apps plugin.
It lets you build and edit Power Apps Canvas Apps using natural language, powered by the
Canvas Authoring MCP server and the `.pa.yaml` source format.

**Plugin root:** this repository root

---

## Prerequisites

Before using any canvas app skill, ensure:

1. **.NET 10 SDK** is installed (`dotnet --list-sdks` shows a `10.x` version)
   - Download: https://dotnet.microsoft.com/download/dotnet/10.0
2. **MCP server is configured** — run `$configure-canvas-mcp` at least once
3. **Power Apps Studio** is open with your canvas app and **coauthoring is enabled**
   - Settings → Updates → Coauthoring → ON
4. **Codex has been restarted** after MCP configuration to load the server

---

## Available Skills

| Skill | Command | When to Use |
|-------|---------|-------------|
| Configure MCP | `$configure-canvas-mcp` | First-time setup, changing apps, MCP errors |
| Generate app | `$generate-canvas-app` | Building a new canvas app from scratch |
| Edit app | `$edit-canvas-app` | Modifying an existing canvas app |
| Add data source | `$add-data-source` | Adding SharePoint / Dataverse / SQL / Excel etc. |

**Invoke explicitly** with the skill name, or describe what you want and Codex will match the skill automatically.

---

## How the Workflow Works

### Generating a New App (`$generate-canvas-app`)

1. **Planner phase** — Codex discovers available controls, connectors, and data sources
   via the MCP server, then presents a screen-by-screen plan for your approval.
   You must reply "approved" before generation begins.
2. **Builder phase** — Codex generates `.pa.yaml` files for each screen, one at a time.
3. **Validation phase** — Codex calls `compile_canvas` and auto-fixes any errors.
4. **Summary** — you get a clean compiled set of `.pa.yaml` files, ready to sync to Studio.

### Editing an Existing App (`$edit-canvas-app`)

1. Codex syncs the current app from Studio via `sync_canvas`
2. Simple edits (1 screen, 1–2 controls) are applied directly
3. Complex edits go through a plan → approval → edit → validate cycle

---

## Agent Architecture

Skills delegate work to specialist agent definitions in `agents/`. These are NOT user-invocable
directly — they are read by skills and their instructions are followed inline.

| Agent File | Called By | Role |
|------------|-----------|------|
| `agents/canvas-app-planner.md` | `generate-canvas-app` | Discovers resources, designs, gets approval, writes plan doc |
| `agents/canvas-screen-builder.md` | `generate-canvas-app` | Implements one screen's `.pa.yaml` |
| `agents/canvas-edit-planner.md` | `edit-canvas-app` | Plans complex edits, gets approval, writes edit plan doc |
| `agents/canvas-screen-editor.md` | `edit-canvas-app` | Applies edits to one screen's `.pa.yaml` |

---

## MCP Server Tools

The `canvas-authoring` MCP server (configured in `~/.codex/config.toml`) exposes:

| Tool | Description |
|------|-------------|
| `compile_canvas` | Validates `.pa.yaml` files against the Power Apps authoring service |
| `sync_canvas` | Pulls current coauthoring session state into local `.pa.yaml` files |
| `list_controls` | Lists all available Power Apps controls |
| `list_apis` | Lists all available connectors in the current session |
| `list_data_sources` | Lists all data sources connected to the current session |
| `describe_control` | Gets full property schema for a specific control |
| `describe_api` | Gets full operation schema for a specific connector |
| `get_data_source_schema` | Gets column names and Power Fx types for a data source |

---

## Reference Documents

Both documents are in `references/` and are read by agents at runtime:

- `references/TechnicalGuide.md` — `.pa.yaml` syntax, control patterns, Power Fx conventions
- `references/DesignGuide.md` — aesthetic guidelines, layout strategies, design anti-patterns

---

## Directory Layout

```
canvas-apps-plugin-codex/
├── AGENTS.md                         ← This file (Codex reads at startup)
├── README.md                         ← Usage guide
├── .codex-plugin/
│   └── plugin.json                  ← Codex plugin manifest
├── .agents/
│   └── skills/
│       ├── configure-canvas-mcp/
│       │   └── SKILL.md             ← Codex MCP config skill
│       ├── generate-canvas-app/
│       │   └── SKILL.md             ← App generation skill
│       ├── edit-canvas-app/
│       │   └── SKILL.md             ← App editing skill
│       └── add-data-source/
│           └── SKILL.md             ← Data source addition skill
├── skills/                          ← Plugin-exposed `$...` skill entrypoints
├── agents/
│   ├── canvas-app-planner.md        ← Planner agent (read by generate skill)
│   ├── canvas-screen-builder.md     ← Screen builder agent (read by generate skill)
│   ├── canvas-edit-planner.md       ← Edit planner agent (read by edit skill)
│   └── canvas-screen-editor.md      ← Screen editor agent (read by edit skill)
└── references/
    ├── TechnicalGuide.md            ← YAML syntax and Power Fx reference
    └── DesignGuide.md               ← Design and aesthetic guidelines
```

---

## Troubleshooting

**MCP tools not available (e.g., `list_controls` fails)**
1. Confirm Codex was restarted after running `$configure-canvas-mcp`
2. Confirm Power Apps Studio is open and coauthoring is enabled
3. Run `$configure-canvas-mcp` again with a fresh Studio URL

**`compile_canvas` errors that won't clear**
- Check that `dotnet --list-sdks` shows a `10.x` SDK
- Ensure the working directory path is absolute, not relative
- Re-run `$configure-canvas-mcp` — the session may have expired

**`sync_canvas` returns empty files**
- The coauthoring session may have timed out — save and reopen the app in Studio
- Ensure the ENV_ID and APP_ID in `~/.codex/config.toml` match the currently open app

**Changes not appearing in Power Apps Studio**
- `compile_canvas` syncs validated YAML back to the studio session automatically
- If not visible, toggle coauthoring off and back on in Studio settings
