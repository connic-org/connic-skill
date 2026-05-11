# Connic Skill

A reusable Agent Skill that teaches AI coding agents how to work with Connic — agents, tools, connectors, the composer SDK, and the platform.

This repo follows the open [SKILL.md](https://github.com/anthropics/skills) standard, so it works across every agent that implements it: Claude Code, Cursor, Codex CLI, GitHub Copilot, Windsurf, Gemini, and others. The same `skills/connic/` directory is the source of truth for all distribution channels below.

## Install

### Any agent (recommended)

The [`npx skills` CLI](https://github.com/vercel-labs/skills) installs the skill for whichever agents you have configured locally:

```bash
npx skills add connic-org/connic-skill
```

Run `npx skills add --help` for per-agent install flags and CI-friendly options.

### Claude Code (plugin marketplace)

This repo is also a [Claude Code plugin marketplace](https://code.claude.com/docs/en/plugin-marketplaces). Inside Claude Code:

```
/plugin marketplace add connic-org/connic-skill
/plugin install connic@connic
```

Update with `/plugin marketplace update connic` whenever a new version is released.

### Manual install

If you'd rather drop the files in by hand, copy `plugins/connic/skills/connic/` into your agent's skills directory:

| Agent | Destination |
| --- | --- |
| Claude Code (user) | `~/.claude/skills/connic/` |
| Claude Code (project) | `.claude/skills/connic/` |
| Cursor | `.cursor/skills/connic/` |
| Codex CLI | `.codex/skills/connic/` |
| Generic | `.agents/skills/connic/` |

The skill works the same regardless of host.

## What it does

Activates whenever a developer is working in a Connic project (anything with a `.connic` file or an `agents/*.yaml` pattern) or asks anything about Connic. The skill teaches the agent:

- The on-disk project layout (`agents/`, `tools/`, `middleware/`, `hooks/`, `schemas/`, `guardrails/`, `tests/`)
- How agent YAML works — every field, every default, every gotcha
- How to write Python tools, middleware, hooks, and custom guardrails — with the **exact** signatures the runtime expects (so generated code actually runs)
- The full predefined-tool catalogue (`db_*`, `query_knowledge`, `trigger_agent`, `web_search`, etc.) and how to wrap them in purpose-driven custom tools
- All eleven connectors (cron, email, kafka, mcp, postgres, s3, sqs, stripe, telegram, webhook, websocket) with correct directions and payload shapes
- The `connic` CLI — real flags only, no fabricated ones
- The dashboard concepts (environments, deployment, observability, KB, DB, judges, approvals, A/B testing, Bridge, REST API)
- Best practices the docs recommend: wrap predefined tools, ship with guardrails, write tests as deploy gates

## Why this exists

Connic is a fast-moving, code-first agent platform — the SDK, the composer features, and the connector catalogue ship updates regularly. This skill is a tight, source-checked reference written directly from the live docs and the `connic-composer-sdk` source, so your AI agent always works against the **current** shape of Connic instead of a stale snapshot from its training data. The result is generated code that runs the first time: real YAML keys, real CLI flags, real predefined-tool signatures, and the conventions the Connic team actually recommends.

Internal evals show **100% vs 56%** pass-rate against a no-skill baseline (delta +44 pp) across six representative tasks. See [evals/](evals/) for the prompts.

## Repository layout

```
.
├── README.md
├── .claude-plugin/
│   └── marketplace.json               # Claude Code marketplace catalog
├── plugins/
│   └── connic/                        # the plugin itself
│       ├── .claude-plugin/
│       │   └── plugin.json            # plugin manifest
│       └── skills/
│           └── connic/
│               ├── SKILL.md           # entry point — always loaded
│               └── references/        # eight on-demand topic files
└── evals/
    └── evals.json                     # the six prompts used to validate the skill
```

`npx skills` and the Claude Code plugin loader both read the same `plugins/connic/skills/connic/` directory — no content duplication.

## Updating

```bash
# Cross-agent
npx skills update connic

# Claude Code
/plugin marketplace update connic
```

## Contributing

The entry point is `skills/connic/SKILL.md`; on-demand reference files are in `skills/connic/references/`. PRs welcome — please keep the existing style: terse, example-led, no marketing prose, every claim checked against the docs or SDK source.

## License

Apache-2.0 — same as the Connic Composer SDK.
