---
name: odoo-commit
description: >
  Generate and validate Odoo-style git commit messages following official Odoo guidelines.
  Use this skill whenever the user asks to write a commit message, stage changes, review what
  to commit, or says things like 'help me commit', 'write a commit message', 'how should I commit
  this', 'generate commit message', or 'what tag should I use' in an Odoo project context. Always
  apply the [TAG] module short description format with a full WHY-focused description body.
---

Expert Odoo git commit message writer. Source: https://www.odoo.com/documentation/18.0/contributing/development/git_guidelines.html

When asked to generate a commit, **always run `git diff --stat` and `git status` first** to understand what changed.

---

## Commit Message Structure

```
[TAG] module: short description (≤ 50 chars)

Full description explaining WHY the change is being made.
Focus on PURPOSE, not on what the diff shows.
Only explain WHAT if a technical decision was made — then explain WHY.

task-123        (related Odoo task)
Fixes #123      (closes GitHub issue)
Closes #123     (closes GitHub PR)
opw-123         (related OPW ticket)
```

---

## All Official Tags

| Tag | When to use |
|-----|-------------|
| `[FIX]` | Bug fix — correcting broken behavior |
| `[ADD]` | Adding a new module or major new feature |
| `[IMP]` | Improvement — incremental enhancement to existing feature |
| `[REF]` | Refactoring — heavy rewrite, no behavior change |
| `[REV]` | Reverting a previous commit |
| `[MOV]` | Moving files only — use `git mv`, do NOT change content |
| `[REM]` | Removing code, files, or features |
| `[REL]` | Release commit — new major/minor stable version |
| `[MERGE]` | Merge commit — forward port or multi-commit feature |
| `[CLA]` | Signing the Odoo Contributor License |
| `[I18N]` | Translation/internationalization changes |

> **Most common in custom development:** `[FIX]`, `[ADD]`, `[IMP]`, `[REF]`, `[MOV]`

---

## Module Name Rules

- Use the **technical name** (e.g. `sale_order_custom`, not "Sales Custom")
- Multiple modules → list them: `module1, module2`
- Cross-module change → use `various`

```
✅  [IMP] sale_order_custom: add approval workflow for large orders
✅  [FIX] account, sale: fix currency rounding on reconciliation
✅  [REF] various: migrate legacy widgets to OWL components
❌  [IMP] Sales Custom: improvements
```

---

## Header Rules (short description)

- Max ~50 characters
- Must complete: **"If applied, this commit will `<header>`"**
- No vague words like "bugfix", "improvements", "update"

```
✅  add approval workflow for large sale orders
✅  prevent duplicate enrollment on same training course
✅  track fuel consumption per vehicle in fleet module
❌  bugfix
❌  improvements
❌  update models
```

---

## Full Description Rules

**Focus on WHY, not WHAT.** The diff already shows what changed.

### ✅ Good

```
Users were able to enroll in the same training course multiple times,
causing duplicate attendance records and incorrect completion reports.
Added SQL constraint + Python validation to block duplicates at both
the ORM and DB levels. Chose SQL constraint over Python-only because
it's enforced even during bulk imports.

task-456
```

### ❌ Bad

```
Added unique constraint on hr.training.enrollment model.
Modified _check_enrollment method.
Updated views to show error message.
```

---

## References

```
task-123          # Odoo task
Fixes #456        # closes a GitHub issue
Closes #789       # closes a GitHub PR
opw-321           # OPW support ticket
```

---

## Git Workflow

```bash
# 1. Check what changed
git status
git diff

# 2. Stage only related changes (one module at a time)
git add addons/your_module/

# 3. Commit
git commit -m "[IMP] your_module: short description

Why this change is needed, context, and any decisions made.

task-123"

# 4. Multi-module: use separate commits
git add addons/module_a/
git commit -m "[FIX] module_a: fix X"

git add addons/module_b/
git commit -m "[FIX] module_b: fix Y"
```

---

## Real Odoo Examples

```
[REF] models: use parent_path to implement parent_store

Replaces the former modified preorder tree traversal (MPTT) with
parent_left/parent_right fields. The new implementation is simpler,
more performant and avoids locking issues during concurrent writes.
```

```
[FIX] account: prevent negative balance on reconciliation

The reconciliation wizard allowed partial reconciliation that resulted
in a negative outstanding balance. This happened because the residual
amount check was done before currency rounding was applied.
Added rounding before the balance check.

Fixes #12345
```

```
[IMP] sale_order_custom: add manager approval for orders > 10000

Sales managers requested a mandatory approval step for large orders
to prevent unauthorized high-value commitments. The threshold is
configurable per company via res.config.settings.

task-789
```

---

## When Generating a Commit

1. Run `git diff --stat` and `git status` to see what changed
2. Pick the correct `[TAG]` based on change nature
3. Use the technical module name from `__manifest__.py`
4. Write a header ≤ 50 chars that completes "If applied, this commit will…"
5. Write a WHY-focused description (3+ sentences)
6. Add references if the user mentions a task/issue number
7. Warn if changes span multiple modules and suggest splitting

---

## Common Mistakes to Avoid

| Mistake | Correct approach |
|---------|-----------------|
| `[IMP] module: update` | `[IMP] module: add bulk import validation for orders` |
| `[FIX] module: fix bug` | `[FIX] module: fix VAT computation on invoices with discount` |
| Using functional module name | Use `sale_order_custom`, not "Sales Order Custom" |
| One giant commit for multiple modules | Split into one commit per module |
| WHY-free description | Always explain the business context |
