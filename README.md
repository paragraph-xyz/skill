# Paragraph Agent Skills

[Agent Skills](https://agentskills.io) for [Paragraph](https://paragraph.com) — lets AI agents manage posts, publications, subscribers, and coins.

## Skills

| Skill | Description | Install required |
|-------|-------------|-----------------|
| [paragraph-cli](paragraph-cli/SKILL.md) | CLI and MCP server | Yes (`npm install -g @paragraph-com/cli`) |
| [paragraph-api](paragraph-api/SKILL.md) | REST API and TypeScript SDK | No |

## Install

Install all skills:

```bash
npx skills add paragraph-xyz/skill
```

Install a specific skill:

```bash
npx skills add paragraph-xyz/skill --skill paragraph-api
npx skills add paragraph-xyz/skill --skill paragraph-cli
```

Or manually copy the `SKILL.md` file into your agent's skills directory:

| Agent | Path |
|-------|------|
| Claude Code | `~/.claude/skills/<skill-name>/SKILL.md` |
| Cursor | `.cursor/skills/<skill-name>/SKILL.md` |
| Any agent | Check your agent's skill directory |

## MCP Server

For MCP-compatible clients, you can also use the hosted Paragraph MCP server — no CLI or API key required:

```bash
claude mcp add paragraph --transport http https://mcp.paragraph.com/mcp
```

## Links

- [Paragraph](https://paragraph.com)
- [API reference & playground](https://paragraph.com/docs/api-reference/)
- [CLI on npm](https://www.npmjs.com/package/@paragraph-com/cli)
- [SDK on npm](https://www.npmjs.com/package/@paragraph-com/sdk)
- [Agent Skills standard](https://agentskills.io)

## License

MIT
