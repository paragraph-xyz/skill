# Paragraph CLI Skill

An [Agent Skill](https://agentskills.io) for the [Paragraph](https://paragraph.com) CLI — lets AI agents manage posts, publications, subscribers, and coins on Paragraph.

## Install

```bash
npx skills add paragraph-xyz/skill
```

Or manually copy `SKILL.md` into your agent's skills directory:

| Agent | Path |
|-------|------|
| Claude Code | `~/.claude/skills/paragraph-cli/SKILL.md` |
| Cursor | `.cursor/skills/paragraph-cli/SKILL.md` |
| Any agent | Check your agent's skill directory |

## Prerequisites

- [Node.js](https://nodejs.org) 18+
- The Paragraph CLI: `npm install -g @paragraph-com/cli`
- A Paragraph API key (get one at [paragraph.com](https://paragraph.com))

## What it does

This skill teaches AI agents how to use the Paragraph CLI to:

- **Create and publish posts** — draft, update, publish, archive, delete
- **Manage subscribers** — list, add, import from CSV
- **Search content** — find posts and blogs
- **Work with coins** — get info, holders, quotes
- **Handle auth** — login, logout, verify credentials

The skill includes working agreements (always use `--json`, never publish without approval, etc.) and common patterns for chaining commands.

## Links

- [Paragraph CLI on npm](https://www.npmjs.com/package/@paragraph-com/cli)
- [Paragraph CLI repo](https://github.com/paragraph-com/paragraph-cli)
- [Paragraph](https://paragraph.com)
- [Agent Skills standard](https://agentskills.io)

## License

MIT
