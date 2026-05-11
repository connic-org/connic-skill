# Connic Skill

A reusable Agent Skill that teaches AI coding agents how to work with Connic — agents, tools, connectors, the composer SDK, and the platform.

This skill follows the open [SKILL.md](https://github.com/anthropics/skills) standard, so it works across every agent that implements it: Claude Code, Cursor, Codex CLI, GitHub Copilot, Windsurf, Gemini, and others — not just Claude.

## Install

The fastest way, for any supported agent:

```bash
npx skills add connic-org/connic-skill
```

The [`npx skills` CLI](https://github.com/vercel-labs/skills) auto-discovers `SKILL.md` at the repo root and installs it for whichever agents you have configured locally. Run `npx skills add --help` for per-agent install flags.

### Manual install (Claude Code)

If you'd rather drop the files in by hand:

```bash
git clone https://github.com/connic-org/connic-skill ~/.claude/skills/connic
```

Restart Claude Code; the skill activates the next time you ask anything related to Connic.

### Manual install (other agents)

| Agent | Skills directory |
| --- | --- |
| Claude Code | `~/.claude/skills/<name>/` or `.claude/skills/<name>/` (project) |
| Cursor | `.cursor/skills/<name>/` |
| Codex CLI | `.codex/skills/<name>/` |
| Generic | `.agents/skills/<name>/` |

Copy the contents of this repo into the appropriate directory. The skill works the same regardless of host.

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

Connic is niche, and most foundation models hallucinate when asked about it — they invent TypeScript SDKs, fictional CLI flags, `slack` connectors, and `postgres.query` tools. This skill is a tight reference written from the actual docs and SDK source so the agent stops guessing.

Internal evals show **100% vs 56%** pass-rate against a no-skill baseline (delta +44 pp) across six representative tasks: see [evals/](evals/) for the prompts and `connic-skill-workspace/` (sibling repo) for the iteration results.

## Updating

```bash
npx skills update connic-skill          # update one
npx skills update                       # interactive update of all
```

## Contributing

The skill lives in `SKILL.md` (the always-loaded entry point) plus eight reference files in `references/` that are loaded on demand. PRs welcome — please keep the existing style: terse, example-led, no marketing prose, every claim checked against the docs or SDK source.

## License

Apache-2.0 — same as the Connic Composer SDK.
