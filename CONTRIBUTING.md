# Contributing to odoo-dev-plugin

Thanks for helping improve Odoo developer skills for Copilot CLI and Claude Code!

## What We Accept

- **Skill updates** — fixing incorrect or outdated knowledge
- **New version coverage** — adding v17/v18/v19 diffs where missing
- **New skills** — well-scoped Odoo development topics not yet covered
- **Bug fixes** — wrong syntax, deprecated APIs, broken examples

## Source Verification Policy

**All skill content must be verifiable against Odoo source code.**

- Cite the Odoo version (e.g., `v18.0`)
- Note the module and file path when relevant (e.g., `mail/models/mail_thread.py`)
- Do not add content based on blog posts, tutorials, or guesswork
- If you can't verify from source, mark it clearly as `# unverified`

## Skill Format

Each skill lives in `skills/<name>/SKILL.md`. Follow the existing structure:

```
# Skill Name

## Overview
Brief description of what this skill covers.

## Version Matrix
| Feature | v17 | v18 | v19 |
|---------|-----|-----|-----|
| ...     | ... | ... | ... |

## Core Concepts
...

## v17.0
...

## v18.0 Changes
...

## v19.0 Changes
...

## Common Mistakes
...
```

Rules:
- Use `## v17.0`, `## v18.0 Changes`, `## v19.0 Changes` as section headers
- Include a version matrix table for any feature that changed across versions
- Include a **Common Mistakes** section
- Code blocks must specify language (` ```python `, ` ```xml `, ` ```js `)
- Keep examples minimal but complete — no pseudocode

## Adding a New Skill

1. Create `skills/<your-skill-name>/SKILL.md`
2. Add `@./skills/<your-skill-name>/SKILL.md` to `AGENTS.md`
3. Add a row to the skills table in `README.md`
4. Open a PR with your changes

## Updating an Existing Skill

1. Edit the relevant `skills/<name>/SKILL.md`
2. In your PR description, cite sources (module/file/version)

## PR Process

1. Fork the repo
2. Create a branch: `git checkout -b skill/your-topic` or `fix/your-fix`
3. Make changes following the format above
4. Open a PR against `main`
5. Fill in the PR template
6. Address review comments — all conversations must be resolved before merge

## Commit Style

Use [Conventional Commits](https://www.conventionalcommits.org/):

```
feat: add odoo-portal skill with v17/v18/v19 coverage
fix: correct ir.cron next_call field name in v18
docs: update migration skill with OCA upgrade-path notes
```

## Questions

Open a [GitHub Issue](https://github.com/halim-23/odoo-dev-plugin/issues) with the `question` label.
