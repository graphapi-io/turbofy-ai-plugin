# Turbofy Plugin for Claude Code

A Claude Code plugin that wires up the Turbofy MCP server and ships three skills covering the Turbofy app-development workflow.

## What's in the box

- **MCP server** (`turbofy`) — registered via [`.mcp.json`](.mcp.json). Runs `npx -y @turbofy-ai/mcp@latest`, so the latest published MCP is fetched on demand.
- **Three skills** auto-trigger based on what you're working on:
  - **`turbofy-apps`** — Apps CMS data model + `Turbofy_app_*` / `Turbofy_workspace_*` workflow (pull → edit → push, file layout under `~/.turbofy/workspaces/<env>/<wsId>/apps/<appId>/`, block-instance editing).
  - **`turbofy-blocks`** — writing block-type React components (`block-types/<Name>/index.tsx`): `IBuildingBlockProps`, `config.copies`, client-side hooks from `@/api`, navigation with `@/navigation`, UI/UX/accessibility rules.
  - **`turbofy-dynamic-fields`** — server-side dynamic-field JavaScript (`defaultConfig`, `defaultDynamicData`, etc.): `$$self`, `$$args`, the `$$std` API, reserved `dynamicArgs` keys.

## Installing

You need a Claude Code marketplace to install from. Once this repo is hosted somewhere (e.g. on GitHub) and listed in a `marketplace.json`, install with:

```bash
/plugin marketplace add <your-org>/<marketplace-repo>
/plugin install turbofy
```

Then restart your Claude Code session (or run `/reload-plugins`). The `turbofy` MCP server will connect automatically and the three skills will be available for auto-trigger or manual invocation (`/turbofy:turbofy-apps`, etc.).

## Local development

To iterate on the plugin without publishing:

```bash
claude --plugin-dir /path/to/turbofy-plugin
```

Inside the session:

```text
/mcp                       # confirm the turbofy MCP is connected
/turbofy:turbofy-apps      # manually invoke a skill
```

After editing files here, run `/reload-plugins` to pick up changes.

## Layout

```
turbofy-plugin/
├── .claude-plugin/
│   └── plugin.json              # manifest (name, version, description)
├── .mcp.json                    # registers the turbofy MCP server
├── skills/
│   ├── turbofy-apps/SKILL.md
│   ├── turbofy-blocks/SKILL.md
│   └── turbofy-dynamic-fields/SKILL.md
└── README.md
```

## MCP source

The MCP server is published from the [`my.graphapi.io`](https://github.com/) monorepo at `packages/mcp/` as the npm package `@turbofy-ai/mcp`. The plugin pins to `@latest`, so users always get the most recent release; bump the MCP's `package.json` version + publish to push updates without touching the plugin.

### Alpha environment (not yet supported)

The MCP currently always runs against the `prod` environment. To run against `alpha`, the MCP needs a small change to honor an env var (e.g. `TURBOFY_ENV=alpha`) in `bin/mcp-server.ts`. Once that lands, the plugin's `.mcp.json` can register a second `turbofy-alpha` server alongside `turbofy`.

## Skill content

The three skills are ported from the `apps-manager` OpenCode agent that previously lived at `.opencode/agents/apps-manager.md` in the `my.graphapi.io` monorepo. Tool references have been updated to match the current `@turbofy-ai/mcp` tool surface (e.g. `Turbofy_data_*` on `CmsOfTypeEnum.*` replaces the per-entity tools the old MCP exposed).
