# Jamdesk Skills

AI agent skills for [Jamdesk](https://jamdesk.com) documentation workflows.

## Installation

Install skills using the [skills CLI](https://github.com/vercel-labs/add-skill):

```bash
# Install all skills
npx skills add jamdesk/skills

# Install a specific skill
npx skills add jamdesk/skills --skill update-jamdesk

# Install globally (available in all projects)
npx skills add jamdesk/skills --skill update-jamdesk -g
```

## Available Skills

| Skill | Description |
|-------|-------------|
| [update-jamdesk](./skills/update-jamdesk) | Automatically updates Jamdesk documentation when code changes |

## Supported Agents

- Claude Code
- Cursor
- Codex
- Windsurf
- OpenCode
- And more...

Run `npx skills add --help` for the full list.

## Documentation

- [Jamdesk Docs](https://jamdesk.com/docs)
- [Automated Doc Updates Guide](https://jamdesk.com/docs/development/ai-documentation)

## License

MIT
