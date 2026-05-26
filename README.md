# Turbofy Plugin

A plugin for Codex, Claude Code, and Cursor that wires up the Turbofy MCP server and ships four skills covering the Turbofy app-development workflow.

## What's in the box

- **MCP server** (`turbofy`) — runs `npx -y @turbofy-ai/mcp@latest`, so the latest published MCP is fetched on demand.
- **Four skills** auto-trigger based on what you're working on:
  - **`turbofy-platform`** — orientation skill and entry point: the platform mental model (org → workspace → schema → apps → pages → blocks → dynamic fields), workspaces & environments (`alpha` / `prod`), the `~/.turbofy/workspaces/<env>/<wsId>/` root, the full MCP tool surface + core rules, the `Turbofy_workspace_*` schema workflow, and the data-builder DSL.
  - **`turbofy-apps`** — Apps CMS data model + `Turbofy_app_*` workflow (pull → edit → push, file layout under `apps/<appId>/`, localization round-trip, block-instance editing, macros).
  - **`turbofy-blocks`** — writing block-type React components (`block-types/<Name>/index.tsx`): `IBuildingBlockProps`, `config.copies`, client-side hooks from `@/api`, navigation with `@/navigation`, UI/UX/accessibility rules.
  - **`turbofy-dynamic-fields`** — server-side dynamic-field JavaScript (`defaultConfig`, `defaultDynamicData`, etc.): `$$self`, `$$args`, the `$$std` API, reserved `dynamicArgs` keys.

## Installing

This repo is both the **plugin** and its **marketplace** (single-repo setup for Codex, Claude Code, and Cursor).

### Codex

Codex reads the marketplace catalog at [`.agents/plugins/marketplace.json`](.agents/plugins/marketplace.json). That catalog points to the Codex plugin under [`plugins/turbofy/`](plugins/turbofy/), whose manifest lives at [`plugins/turbofy/.codex-plugin/plugin.json`](plugins/turbofy/.codex-plugin/plugin.json). Installing the plugin gives Codex both the Turbofy skills and the `turbofy` MCP server.

Use the Codex plugin UI to add this repository as a marketplace:

| Field | Value |
| --- | --- |
| Source | `git@github.com:graphapi-io/turbofy-ai-plugin.git` or `https://github.com/graphapi-io/turbofy-ai-plugin.git` |
| Git ref | `main` |
| Sparse paths | Leave empty |

For local development, point Codex at this repo as a local marketplace source, then restart or reload plugins so the skills and MCP server are picked up.

### Claude Code

```bash
/plugin marketplace add <your-org>/turbofy-plugin
/plugin install turbofy@turbofy
```

Then `/reload-plugins` (or restart the session). The `turbofy` MCP connects automatically and the three skills are available for auto-trigger or manual invocation (`/turbofy:turbofy-apps`, etc.).

### Cursor

Cursor reads the catalog at [`.cursor-plugin/marketplace.json`](.cursor-plugin/marketplace.json) and the plugin manifest at [`.cursor-plugin/plugin.json`](.cursor-plugin/plugin.json). Add the marketplace via Cursor's plugin UI (or however the current Cursor build prefers), and Cursor will pick up the `turbofy` plugin + its MCP server and skills.

### Manual MCP setup (no plugin install)

If you'd rather just wire up the MCP without using the plugin system, copy the content of [`mcp.json`](mcp.json) into your client's MCP config:

- Claude Code: `~/.claude.json` or project `.mcp.json`
- Cursor: `~/.cursor/mcp.json` or project `.cursor/mcp.json`

## Local development

To iterate without publishing:

```bash
# Claude Code
claude --plugin-dir /path/to/turbofy-plugin
```

Inside the session:

```text
/mcp                       # confirm the turbofy MCP is connected
/turbofy:turbofy-apps      # manually invoke a skill
```

After editing files here, run `/reload-plugins` to pick up changes.

To validate the manifests:

```bash
claude plugin validate /path/to/turbofy-plugin
```

## Layout

```
turbofy-plugin/
├── .agents/
│   └── plugins/
│       └── marketplace.json        # Codex marketplace catalog
├── .claude-plugin/
│   ├── plugin.json                 # Claude Code plugin manifest
│   └── marketplace.json            # Claude Code marketplace catalog
├── .cursor-plugin/
│   ├── plugin.json                 # Cursor plugin manifest
│   └── marketplace.json            # Cursor marketplace catalog
├── .mcp.json                       # Claude Code MCP server config
├── mcp.json                        # Cursor MCP server config (same content)
├── plugins/
│   └── turbofy/                    # Codex plugin root
│       ├── .codex-plugin/
│       │   └── plugin.json         # Codex plugin manifest
│       ├── .mcp.json               # Codex MCP server config
│       └── skills/                 # Codex skill copies
├── skills/                         # SKILL.md files (cross-tool)
│   ├── turbofy-platform/SKILL.md
│   ├── turbofy-apps/SKILL.md
│   ├── turbofy-blocks/SKILL.md
│   └── turbofy-dynamic-fields/SKILL.md
└── README.md
```

The skills are the only meaningful shared content; the rest is small bookkeeping per ecosystem. Keep `mcp.json` and `.mcp.json` in sync if you change either one.

## MCP source

The MCP server is published from the [`my.graphapi.io`](https://github.com/) monorepo at `packages/mcp/` as the npm package `@turbofy-ai/mcp`. The plugin pins to `@latest`, so users always get the most recent release; bump the MCP's `package.json` version + publish to push updates without touching the plugin.

### Alpha environment (not yet supported)

The MCP currently always runs against the `prod` environment. To run against `alpha`, the MCP needs a small change to honor an env var (e.g. `TURBOFY_ENV=alpha`) in `bin/mcp-server.ts`. Once that lands, the MCP configs can register a second `turbofy-alpha` server alongside `turbofy`.

## Skill content

The three skills are ported from the `apps-manager` OpenCode agent that previously lived at `.opencode/agents/apps-manager.md` in the `my.graphapi.io` monorepo. Tool references have been updated to match the current `@turbofy-ai/mcp` tool surface (e.g. `Turbofy_data_*` on `CmsOfTypeEnum.*` replaces the per-entity tools the old MCP exposed).

SKILL.md frontmatter uses only `name` and `description` (the universal subset of the format). Both Claude Code and Cursor consume the same files unchanged.
