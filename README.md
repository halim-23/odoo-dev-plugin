# odoo-dev-plugin

Comprehensive **Odoo development skills** for [GitHub Copilot CLI](https://docs.github.com/copilot/concepts/agents/about-copilot-cli) and [Claude Code](https://docs.anthropic.com/en/docs/claude-code).

Covers **Odoo 17.0, 18.0, 19.0** — Community & Enterprise — with source-verified, version-accurate knowledge.

## Install

### GitHub Copilot CLI

```bash
copilot plugin install github:halim-23/odoo-dev-plugin
```

### Claude Code

```bash
# Step 1 — add this repo as a marketplace (one time)
claude plugin marketplace add halim-23/odoo-dev-plugin

# Step 2 — install the plugin
claude plugin install odoo-dev
```

**Update later:**
```bash
claude plugin update odoo-dev
```

## Skills

| Skill | Covers |
|-------|--------|
| `/odoo-model` | Fields, ORM, compute/inverse, constrains, dynamic domains, v19 `Domain` class |
| `/odoo-view` | Form/list/kanban/search arch, `attrs`, chatter, widgets, deprecations |
| `/odoo-module` | Manifest, `ir.cron` (v17/v18/v19 diffs), data/demo XML, hooks, tests |
| `/odoo-controller` | HTTP routes, JSON-RPC, auth types, CORS, session |
| `/odoo-wizard` | `TransientModel`, wizard flow, multi-step, return actions |
| `/odoo-security` | `ir.model.access`, `ir.rule`, groups, `sudo()`, portal |
| `/odoo-report` | QWeb PDF, XLSX, `ir.actions.report`, custom parsers |
| `/odoo-js` | OWL components, field widgets, `view_widgets`, custom view types, `doAction`, hooks |
| `/odoo-migration` | OCA migration scripts, `odoo.sh`, on-prem upgrade, v17→v18→v19 |
| `/odoo-debug` | Logging, shell, profiler, query count, `--dev` modes |
| `/odoo-mail-template` | XML email templates v17/v18/v19, `t-out`, QWeb, jinja2 |

## Usage

Use skill name in prompt:

```
Use /odoo-model to create a product.template inheritance with computed price field
```

Or Copilot auto-selects the relevant skill based on your prompt.

## Version Coverage

- **v17.0** Community & Enterprise
- **v18.0** Community & Enterprise
- **v19.0** Community & Enterprise

All skills include version-specific notes, deprecation tables, and migration guidance.
