# Jamdesk Skills

AI agent skills for [Jamdesk](https://jamdesk.com) documentation workflows. These skills help AI coding assistants automatically keep your documentation in sync with your code.

## What is Jamdesk?

[Jamdesk](https://jamdesk.com) is a documentation platform that transforms MDX (Markdown + React components) into polished documentation websites. Write in Markdown, push to GitHub, and deploy globally in seconds.

**Key features:**
- MDX-based workflow with 25+ built-in components
- Three professional themes (Jam, Nebula, Pulsar)
- AI-powered search and LLM-ready output (`llms.txt`)
- Built-in analytics and OpenAPI support
- Custom domains with SSL

## Why Use These Skills?

Documentation often falls out of sync with code. These skills solve that by teaching AI coding assistants how to automatically update your Jamdesk docs when you make changes to APIs, CLI commands, UI components, or configuration options.

**The workflow:**
1. You implement a feature in your codebase
2. Run `/update-jamdesk` in your AI assistant
3. The skill analyzes your changes and creates or updates documentation
4. Review, verify, and commit

No more outdated docs. No more "I'll document it later."

## Installation

Install skills using the [skills CLI](https://github.com/vercel-labs/add-skill):

```bash
# Install a specific skill
npx skills add jamdesk/skills --skill update-jamdesk

# Install globally (available in all projects)
npx skills add jamdesk/skills --skill update-jamdesk -g

# Install all Jamdesk skills
npx skills add jamdesk/skills
```

## Available Skills

| Skill | Description |
|-------|-------------|
| [update-jamdesk](./skills/update-jamdesk) | Automatically updates Jamdesk documentation when code changes |

## Supported AI Coding Assistants

These skills work with any AI coding assistant that supports the skills format:

- **Claude Code** - Anthropic's CLI for Claude
- **Cursor** - AI-first code editor
- **Windsurf** - Codeium's AI IDE
- **Codex** - OpenAI's coding assistant
- **OpenCode** - Open-source AI coding tool
- And 30+ more agents

Run `npx skills add --help` for the full list of supported agents.

## How It Works

The `update-jamdesk` skill teaches your AI assistant to:

1. **Find your docs** - Locates your Jamdesk project via a `.jamdesk-docs-path` config file
2. **Understand changes** - Analyzes what you've built and asks clarifying questions
3. **Write documentation** - Creates or updates MDX pages following Jamdesk conventions
4. **Verify** - Runs `jamdesk validate` and checks for broken links
5. **Commit** - Stages changes and offers to commit/push/create a PR

## Requirements

- A [Jamdesk](https://jamdesk.com) documentation project with `docs.json`
- An AI coding assistant (Claude Code, Cursor, etc.)
- Optional: [Jamdesk CLI](https://jamdesk.com/docs/cli/overview) for local validation

## Documentation

- [Jamdesk Documentation](https://jamdesk.com/docs) - Full platform docs
- [Automated Doc Updates Guide](https://jamdesk.com/docs/development/ai-documentation) - Detailed skill usage
- [MDX Basics](https://jamdesk.com/docs/content/mdx-basics) - Writing content in Jamdesk
- [Components Reference](https://jamdesk.com/docs/components/overview) - Available MDX components

## Contributing

To add a new skill:

1. Create a directory: `skills/<skill-name>/`
2. Add a `SKILL.md` file with YAML frontmatter (`name`, `description`)
3. Write instructions in Markdown
4. Submit a pull request

See [CLAUDE.md](./CLAUDE.md) for detailed guidelines.

## License

MIT
