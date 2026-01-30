# CLAUDE.md

This file provides guidance to Claude Code when working with this repository.

## Overview

This repository contains AI agent skills for [Jamdesk](https://jamdesk.com) documentation workflows. Skills are installed via the `skills` CLI.

## Repository Structure

```
skills/
├── README.md           # Installation instructions
├── LICENSE             # MIT
├── CLAUDE.md           # This file
└── skills/
    └── update-jamdesk/
        └── SKILL.md    # Skill definition
```

## Adding a New Skill

1. Create a directory: `skills/<skill-name>/`
2. Add `SKILL.md` with required frontmatter:
   ```yaml
   ---
   name: skill-name
   description: Brief description of what this skill does
   ---
   ```
3. Write instructions in markdown below the frontmatter
4. Update `README.md` to list the new skill
5. Commit and push

## Skill Format

Skills follow the [Vercel Labs add-skill format](https://github.com/vercel-labs/add-skill):

- **Required:** `name` and `description` in YAML frontmatter
- **Body:** Markdown instructions for the AI agent
- **Optional:** `scripts/`, `references/`, `assets/` subdirectories

## Testing

Test skill discovery:
```bash
npx skills add jamdesk/skills --list
```

Test installation:
```bash
npx skills add jamdesk/skills --skill <skill-name>
```

## Related

- Main monorepo: `github.com/jamdesk/jamdesk` (internal)
- Documentation: `jamdesk.com/docs/development/ai-documentation`
