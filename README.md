# Threatcl Cloud — Claude Code Plugin

A Claude Code plugin that brings [Threatcl Cloud](https://threatcl.com) into your AI coding workflow. Bundles:

- **Skill** — `threatcl` skill auto-triggers on threat-modeling work and guides the agent through the right commands.
- **Slash commands** — five workflow commands for review, code analysis, drift detection, CI scaffolding, and new-model creation.
- **MCP server** — the Threatcl Cloud MCP server (`threatcl`) for direct read access to your org's threat models, library, and analytics. Authenticated via Claude Code's built-in OAuth flow on first use.

## Install

```
/plugin marketplace add threatcl/claude-plugin
/plugin install threatcl-cloud@threatcl
```

The first time you use a Threatcl Cloud feature, Claude Code will open a browser to complete OAuth — pick your org and you're done.

## Prerequisites

- **Claude Code** — this plugin is Claude Code-specific. Codex and other agents should follow the [agent setup guide](https://threatcl.com/agent-setup.md) instead.
- **`threatcl` CLI** — required for write operations (validate, push, library import, policy create) and local file work. Install:
  - macOS / Linux: `brew install threatcl`
  - Go: `go install github.com/threatcl/threatcl/cmd/threatcl@latest`
  - Releases: <https://github.com/threatcl/threatcl/releases>
- A Threatcl Cloud account at <https://threatcl.com>.

## What's in the box

### Skill

`skills/threatcl/SKILL.md` — the same instruction set you'd get from <https://threatcl.com/threatcl.SKILL.md>, with a short preamble explaining the plugin's MCP/CLI split. Auto-triggers when the agent detects threat-modeling intent.

### Slash commands

| Command | Purpose |
|---|---|
| `/threat-review <model>` | Run a structured security review of a model: unmitigated threats, STRIDE coverage, policy evaluation, library-fit suggestions, prioritized next actions. |
| `/threat-for-code <path-or-diff>` | Analyze code (file, directory, or git diff range) and suggest threats from the org library — plus novel threats worth modeling, with HCL snippets. |
| `/threat-drift [diff-range]` | Detect drift between recent code changes and the documented threat model. Surfaces stale threat assertions, phantom controls, new unmodeled surface, DFD drift, and dependency drift. Defaults to `main...HEAD`. |
| `/threat-ci <flavor>` | Scaffold CI integration. Flavors: `github-actions`, `gitlab-ci`, `pre-commit`. Generates the workflow file with `threatcl cloud validate` on PR and `threatcl cloud policy evaluate` on merge. |
| `/threat-hcl-new <name>` | Scaffold a new HCL threat model file with the cloud backend block pre-populated and a placeholder threat to fill in. |

### MCP server

`.mcp.json` declares the `threatcl` MCP server at the Threatcl Cloud API endpoint. The agent gets read tools automatically (`list_threat_models`, `get_threat_model`, `search`, library lookups, usage analytics). Auth is handled by Claude Code via OAuth 2.1 with PKCE — no tokens to copy around.

For write operations (push, validate, library import, policy edits), the skill falls back to the `threatcl` CLI.

## Architecture

```
┌──────────────────────────────────────────┐
│             Claude Code                  │
│                                           │
│  Skill ──► picks the right tool          │
│                                           │
│  /threat-review  ─┐                      │
│  /threat-for-code │                      │
│  /threat-drift    ┼─► structured prompt  │
│  /threat-ci       │                      │
│  /threat-hcl-new ─┘                      │
└──────────┬─────────────────┬─────────────┘
           │                 │
           ▼                 ▼
   ┌──────────────┐   ┌──────────────┐
   │ MCP server   │   │ threatcl CLI │
   │ (reads, OAuth)│   │ (writes,     │
   │              │   │  local ops)  │
   └──────┬───────┘   └──────┬───────┘
          │                  │
          ▼                  ▼
        Threatcl Cloud (api.threatcl.com)
```

## Configuration

The plugin defaults to the production Threatcl Cloud API. If your org is on the beta endpoint, edit `.mcp.json` after install or set `THREATCL_API_URL` for the CLI.

## License

MIT - see `LICENSE`.

## Links

- Threatcl Cloud — <https://threatcl.com>
- threatcl CLI — <https://github.com/threatcl/threatcl>
- Cloud docs — <https://threatcl.dev/cloud/overview/>
- Issues — <https://github.com/threatcl/claude-plugin/issues>
