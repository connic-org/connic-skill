# Connic Skill

A reusable Agent Skill that teaches AI coding agents how to work with Connic — agents, tools, connectors, Connic MCP, the Composer SDK, and the platform.

This repo follows the open [SKILL.md](https://github.com/anthropics/skills) format. Every installation method below uses `plugins/connic/skills/connic/`.

Installing the full Connic plugin through the ChatGPT/Codex, Claude, or Cursor marketplace also registers the production MCP endpoint. Skill-only installation methods remain standalone and do not modify the client's MCP configuration.

See [AI agent setup](https://connic.co/docs/v1/ai-agent-setup) for client-specific plugin, skill, MCP, and OAuth instructions.

## Install

### Existing Connic project

If the [Connic Composer SDK](https://github.com/connic-org/connic-composer-sdk) is installed, run this from the project root:

```bash
connic skill
```

This installs the skill in both `.agents/skills/connic/` and `.claude/skills/connic/`. For a new project, `connic init <name> --skill` installs both copies while scaffolding.

In an interactive terminal, both CLI paths also detect Codex and Claude Code and ask separately whether to install the full plugin. Declining leaves the skill-only project copies in place.

At the start of each agent session, the skill instructs the agent to run `connic update --check` once before Connic work. The check only reports available updates; the agent must ask before installing one.

### Any agent (recommended)

The [`npx skills` CLI](https://github.com/vercel-labs/skills) installs the skill for whichever agents you have configured locally:

```bash
npx skills add connic-org/connic-skill
```

Run `npx skills add --help` for per-agent install flags and CI-friendly options.

### ChatGPT app (plugin marketplace)

1. Open **Plugins → Add → Add a marketplace**.
2. Enter `connic-org/connic-skill` in **Source**, leave **Git ref** and **Sparse paths** blank, then select **Add marketplace**.
3. Open the Connic marketplace, select Connic, and choose **Install**.
4. Complete the connection prompt, then start a new conversation or Codex task.

### Codex CLI (plugin marketplace)

Install the full plugin when you want the skill and Connic MCP together:

```bash
codex plugin marketplace add connic-org/connic-skill
codex plugin add connic@connic
```

Start a new Codex task after installation, then open `/mcp` to authenticate or inspect the connection registered from `.mcp.json`.

### Claude app (Code)

1. Open a local or SSH **Code** session and select **+ → Plugins → Browse plugins**. If no plugins are installed, select **Add plugins…** directly.
2. In the plugin browser, select **+ Add marketplace → Add from a repository**.
3. Enter `connic-org/connic-skill` in **URL**, select **Sync**, then open Connic and choose **Install**.
4. On the installed Connic plugin page, open **Connectors** and select **Install** next to `connic`.
5. In the **Add custom connector** modal, select **Add**. Back on the same plugin page, select **Connect** and complete browser OAuth. No running Code session or app restart is needed for the connector setup.

### Claude Code (plugin marketplace)

This repo is also a [Claude Code plugin marketplace](https://code.claude.com/docs/en/plugin-marketplaces). Inside Claude Code:

```
/plugin marketplace add connic-org/connic-skill
/plugin install connic@connic
```

The Claude Code plugin automatically registers `https://mcp.connic.co/mcp` from the plugin's `.mcp.json`. Open `/mcp` to complete OAuth or inspect the connection.

Run `/plugin marketplace update connic` to refresh the marketplace installation.

### Cursor (GitHub plugin marketplace)

Enter this in a Cursor Agent chat, then choose a project or user scope:

```text
/add-plugin connic@https://github.com/connic-org/connic-skill
```

The Cursor plugin loads the same skill and registers `https://mcp.connic.co/mcp` from `mcp.json`. Start a new Agent chat, open **Customize**, find the Connic MCP server, and complete OAuth when prompted.

### Manual install

If you'd rather drop the files in by hand, copy `plugins/connic/skills/connic/` into your agent's skills directory:

| Agent | Destination |
| --- | --- |
| Claude Code (user) | `~/.claude/skills/connic/` |
| Claude Code (project) | `.claude/skills/connic/` |
| Cursor | `.cursor/skills/connic/` |
| Codex CLI | `.agents/skills/connic/` |
| Generic | `.agents/skills/connic/` |

All installation methods use the same skill files.

## What it does

Activates whenever a developer is working in a Connic project (anything with a `.connic` file or an `agents/*.yaml` pattern) or asks anything about Connic. The skill teaches the agent:

- The on-disk project layout (`agents/`, `tools/`, `middleware/`, `hooks/`, `schemas/`, `guardrails/`, `tests/`)
- Agent YAML fields, defaults, and validation rules
- How to use EU-hosted `connic/*` models and configured BYOK providers
- Exact signatures for Python tools, middleware, hooks, and custom guardrails
- The predefined-tool catalogue (`db_*`, `retrieval_query`, `trigger_agent`, `web_search`, etc.) and how to wrap them in purpose-driven custom tools
- Built-in connectors (cron, email, kafka, mcp, postgres, s3, sqs, stripe, telegram, webhook, websocket), including directions and payload shapes
- The differences between Connic MCP, agent-side MCP servers, and the MCP connector
- Supported `connic` CLI commands and flags
- The dashboard concepts (Project credit and billing, environments, deployment, observability, Retrieval, DB, judges, approvals, A/B testing, Bridge, REST API)
- Best practices the docs recommend: wrap predefined tools, ship with guardrails, write tests as deploy gates

## Why this exists

This skill gives AI coding agents the supported YAML keys, CLI flags, predefined-tool signatures, and project conventions.

The eval suite covers representative standalone and live Connic MCP behavior. See [evals/](evals/) for the prompts.

## Repository layout

```
.
├── README.md
├── .agents/plugins/
│   └── marketplace.json               # ChatGPT and Codex marketplace catalog
├── .claude-plugin/
│   └── marketplace.json               # Claude app and Claude Code marketplace catalog
├── .cursor-plugin/
│   └── marketplace.json               # Cursor marketplace catalog
├── plugins/
│   └── connic/                        # the plugin itself
│       ├── .mcp.json                  # bundled production Connic MCP connection
│       ├── .claude-plugin/
│       │   └── plugin.json            # Claude Code plugin manifest
│       ├── .codex-plugin/
│       │   └── plugin.json            # Codex plugin manifest
│       ├── .cursor-plugin/
│       │   └── plugin.json            # Cursor plugin manifest
│       ├── assets/
│       │   └── connic-icon.png         # shared plugin icon
│       ├── codex.mcp.json              # Codex MCP connection
│       ├── mcp.json                    # Cursor MCP connection
│       └── skills/
│           └── connic/
│               ├── SKILL.md           # entry point — always loaded
│               └── references/        # on-demand topic references
└── evals/
    └── evals.json                     # prompts used to validate the skill
```

`npx skills`, ChatGPT/Codex, Claude, and Cursor all read the same `plugins/connic/skills/connic/` directory — no content duplication.

## Updating

```bash
# SDK and skill
connic update

# SDK only
connic update --sdk

# Skill only
connic update --skill

# Cross-agent
npx skills update connic

# Claude Code
/plugin marketplace update connic
```

## Contributing

The entry point is `plugins/connic/skills/connic/SKILL.md`; on-demand reference files are in `plugins/connic/skills/connic/references/`. PRs welcome — please keep the existing style: terse, example-led, no marketing prose, every claim checked against the docs or SDK source.

## License

Apache-2.0 — same as the Connic Composer SDK.
