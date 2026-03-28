# Jamdesk Skills

[![License: Apache 2.0](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](./LICENSE)
[![Claude Code](https://img.shields.io/badge/Claude_Code-skills-blueviolet?logo=anthropic)](https://www.jamdesk.com/docs/claude-code)
[![AI Agent](https://img.shields.io/badge/AI_Agent-compatible-green)](https://github.com/jamdesk/skills)

AI agent skills for [Jamdesk](https://jamdesk.com) workflows. These skills teach AI coding assistants to keep documentation in sync with code and handle common developer tasks like screenshot redaction.

## What is Jamdesk?

[Jamdesk](https://jamdesk.com) is a documentation platform that transforms MDX (Markdown + React components) into polished documentation websites. Write in Markdown, push to GitHub, and deploy globally in seconds.

**Key features:**
- MDX-based workflow with 25+ built-in components
- Three professional themes (Jam, Nebula, Pulsar)
- AI-powered search and LLM-ready output (`llms.txt`)
- Built-in analytics and OpenAPI support
- Custom domains with SSL

**New to Jamdesk?** Check out the [documentation](https://jamdesk.com/docs) to get started, or try the [quickstart guide](https://jamdesk.com/docs/quickstart) to deploy your first docs site in minutes.

## Why Use These Skills?

AI coding assistants are capable but generic. Skills give them domain-specific knowledge — how your docs are structured, what tools to use, what steps to follow. Instead of explaining your workflow every time, the skill handles it.

**update-jamdesk** keeps docs in sync with code. Implement a feature, run `/update-jamdesk`, and the skill analyzes your changes, writes documentation, and verifies it. No more "I'll document it later."

**blur-image** redacts sensitive data from screenshots before you share them. Point it at a screenshot and it scans for API keys, tokens, passwords, emails, names, and internal URLs using AI vision — then asks you in plain English what to blur. It handles everything from terminal screenshots with `.env` files to JSON API responses to faces in photos. You describe what to hide in words ("blur the values after the = signs"), and it handles the pixel math, coordinate calibration, and verification automatically.

## Quick Start

```bash
# Install a single skill
npx skills add jamdesk/skills --skill blur-image

# Or install all skills at once
npx skills add jamdesk/skills
```

Then use it in your AI assistant:

```
> blur the sensitive data in screenshot.png
```

For `update-jamdesk`, you'll also need a config file pointing to your docs repo:

```bash
echo "docs_path: ../my-docs" > .jamdesk-docs-path
```

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
| [blur-image](./skills/blur-image) | Detects and blurs sensitive data (API keys, PII, credentials) in screenshots with per-line precision |

## Supported AI Coding Assistants

These skills work with any AI coding assistant that supports the skills format:

- **Claude Code** - Anthropic's CLI for Claude
- **Cursor** - AI-first code editor
- **Windsurf** - Codeium's AI IDE
- **Codex** - OpenAI's coding assistant
- **OpenCode** - Open-source AI coding tool
- And 30+ more agents

Run `npx skills add --help` for the full list of supported agents.

## How They Work

### update-jamdesk

Teaches your AI assistant to find your docs repo, analyze code changes, write MDX pages following Jamdesk conventions, run `jamdesk validate`, and commit. Requires a `.jamdesk-docs-path` config file:

```yaml
# Path to your Jamdesk docs (relative or absolute)
docs_path: ../my-docs
docs_branch: main  # Optional, default: main
```

**Requirements:** A [Jamdesk](https://jamdesk.com) docs project with `docs.json`, an AI coding assistant, and optionally the [Jamdesk CLI](https://jamdesk.com/docs/cli/overview) for local validation.

### blur-image

Detects and blurs sensitive data in screenshots so you can share them safely. The skill:

1. **Scans** the image with AI vision for credentials, PII, tokens, internal URLs, and other sensitive content
2. **Discusses** what it found in plain English — you decide what to blur and what to keep
3. **Calibrates** pixel coordinates using a grid overlay system, so blur regions land exactly on the right text (not one line above or below)
4. **Blurs** with per-line precision — for a config file, it blurs each value individually while keeping the key names readable (`DATABASE_URL=` stays visible, the connection string disappears)
5. **Verifies** by re-reading the output image and checking that every region is fully obscured

Supports terminal screenshots (high-contrast text with double-pass blur), web pages, config files, JSON responses, and non-text content like faces and logos. Can use Gaussian blur (shows a visible smudge indicating redaction) or solid fill (completely removes the content) depending on your preference.

**Requirements:** [ImageMagick 7+](https://imagemagick.org/script/download.php) (`brew install imagemagick` on macOS).

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

Apache 2.0
