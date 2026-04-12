---
name: odoo-migration
description: >
  Odoo migration for versions 17.0, 18.0, 19.0 (Community & Enterprise).
  Full knowledge of: OCA community migration (odoo-migrate CLI, openupgradelib, OpenUpgrade),
  enterprise migrations for odoo.sh (branch-based), on-premises (manual upgrade),
  and Odoo Online/SaaS (managed, no code). Pre/post migration scripts, version upgrade
  paths 16→17→18→19, data transforms, common migration patterns.
  Use when user asks about Odoo migration, upgrade, or invokes /odoo-migration.
---

Expert Odoo migration engineer. Full knowledge: OCA, odoo.sh, on-prem, Odoo Online. v16→v17→v18→v19.

---

## Migration Paths

```
v16 ──► v17 ──► v18 ──► v19
         │        │        │
       CE/EE    CE/EE    CE/EE
```

**Rules:**
- Always upgrade **one major version at a time** — no skipping (v16→v18 not supported)
- Always upgrade on a **database copy** first
- Enterprise and Community: same process, EE adds extra modules
- OCA modules: use OpenUpgrade database migration, then update module code with `odoo-migrate`

---

## Part 1: Community (OCA) Migration

### Tools Overview

| Tool | Purpose | Source |
|------|---------|--------|
| `odoo-migrate` | Updates module Python/XML/JS code to new version syntax | `pip install odoo-migrate` |
| `openupgradelib` | Helpers for migration scripts (rename, merge, delete columns) | `pip install openupgradelib` |
| `OpenUpgrade` | Drop-in Odoo replacement that runs DB migrations for CE | github.com/OCA/OpenUpgrade |

---

### Step 1: Update Module Code with `odoo-migrate`

`odoo-migrate` automates ~80% of syntax changes per version.

```bash
pip install odoo-migrate

# Migrate a single module from v16 to v17
odoo-migrate \
    --directory /path/to/my_module \
    --init-version-name 16.0 \
    --target-version-name 17.0

# Migrate multiple modules
odoo-migrate \
    --directory /path/to/addons_folder \
    --init-version-name 16.0 \
    --target-version-name 17.0 \
    --modules my_module,another_module

# Preview changes without writing (dry run)
odoo-migrate \
    --directory /path/to/my_module \
    --init-version-name 16.0 \
    --target-version-name 17.0 \
    --no-commit \
    --log-level debug
```

**What `odoo-migrate` auto-fixes (v16→v17):**
- `attrs` → inline `invisible`/`required`/`readonly` expressions
- `states` → `invisible` expressions
- `<tree>` → `<list>`
- `t-name="kanban-box"` → `t-name="card"`
- `track_visibility` → `tracking`
- `@api.model` on `create` → `@api.model_create_multi`
- `name_get()` → `_compute_display_name()`
- `colors`/`fonts` → `decoration-*`
- JS `odoo.define` → `/** @odoo-module **/`
- Manifest version bump

**Always review the diff** — `odoo-migrate` is not perfect, especially for complex `attrs`.

---

### Step 2: Write Migration Scripts with `openupgradelib`

```python
# migrations/17.0.1.1.0/pre-migrate.py
import logging
from openupgradelib import openupgrade

_logger = logging.getLogger(__name__)


@openupgrade.migrate()
def migrate(env, version):
    """openupgrade.migrate() decorator handles version check automatically."""
    if not version:
        return
    _rename_columns(env.cr)


def _rename_columns(cr):
    # Safe rename — checks column exists first
    openupgrade.rename_columns(cr, {
        'my_model': [('old_field', 'new_field')],
        'my_other_model': [('old_col', 'new_col')],
    })
```

```python
# migrations/17.0.1.1.0/post-migrate.py
import logging
from openupgradelib import openupgrade

_logger = logging.getLogger(__name__)


@openupgrade.migrate()
def migrate(env, version):
    if not version:
        return
    _merge_selection_values(env.cr)
    _fill_new_field(env)
    _recompute_fields(env)


def _merge_selection_values(cr):
    openupgrade.map_values(
        cr,
        'my_model',
        'state',
        [('old_draft', 'draft'), ('old_done', 'done')],
    )

def _fill_new_field(env):
    env.cr.execute("""
        UPDATE my_model m
        SET new_ref = p.ref
        FROM res_partner p
        WHERE m.partner_id = p.id
          AND m.new_ref IS NULL
    """)

def _recompute_fields(env):
    openupgrade.logged_query(env.cr, """
        UPDATE my_model SET amount_total = 0 WHERE amount_total IS NULL
    """)
    # Force ORM recompute of stored computed fields
    records = env['my.model'].search([])
    records.modified(['line_ids'])
    records.flush_recordset(['amount_total'])
```

### `openupgradelib` Helper Reference

```python
from openupgradelib import openupgrade

# Column operations
openupgrade.rename_columns(cr, {'table': [('old', 'new')]})
openupgrade.drop_columns(cr, [('table', 'column')])
openupgrade.column_exists(cr, 'table', 'column')  # → bool
openupgrade.table_exists(cr, 'table')  # → bool
openupgrade.add_columns(cr, [('table', 'column', 'type', 'default')])

# Table operations
openupgrade.rename_tables(cr, [('old_table', 'new_table')])

# XML ID operations
openupgrade.rename_xmlids(cr, [('old_module.old_id', 'new_module.new_id')])
openupgrade.delete_records_safely_by_xml_id(env, ['module.xml_id'])

# Field value mapping (selection field values)
openupgrade.map_values(cr, 'table', 'column', [('old_val', 'new_val')])

# Module operations
openupgrade.rename_models(cr, [('old.model', 'new.model')])
openupgrade.rename_fields(env, [('my.model', 'old_field', 'new_field', 'my_model')])
# params: model_name, old_field, new_field, table_name

# Logged query (logs SQL for debugging)
openupgrade.logged_query(cr, "UPDATE ...", warn_no_result=True)

# Move data between models
openupgrade.merge_models(cr, 'source.model', 'target.model', 'res_id_mapping')

# Delete obsolete XML records
openupgrade.delete_records_safely_by_xml_id(env, [
    'my_module.old_menu_item',
    'my_module.old_action',
])
```

---

### Step 3: Run OpenUpgrade (Community Database Migration)

OpenUpgrade is a **fork of Odoo CE** that adds pre/post migration scripts for all core modules.

```bash
# 1. Clone OpenUpgrade for target version
git clone https://github.com/OCA/OpenUpgrade.git -b 17.0 /opt/openupgrade-17

# 2. Install dependencies
pip install -r /opt/openupgrade-17/requirements.txt

# 3. Backup production database FIRST
pg_dump mydb > /backup/mydb_before_upgrade_$(date +%Y%m%d).sql
# Also backup filestore:
tar czf /backup/filestore_$(date +%Y%m%d).tar.gz ~/.local/share/Odoo/filestore/mydb/

# 4. Run upgrade on a COPY of the database
createdb mydb_upgrade
pg_restore -d mydb_upgrade /backup/mydb_before_upgrade_YYYYMMDD.sql
# OR
psql -c "CREATE DATABASE mydb_upgrade TEMPLATE mydb"

# 5. Run OpenUpgrade
python /opt/openupgrade-17/odoo-bin \
    -d mydb_upgrade \
    -u all \
    --stop-after-init \
    --load=base,web \
    --logfile=/tmp/upgrade.log \
    2>&1 | tee /tmp/upgrade_output.log

# 6. Check logs for errors
grep -i "error\|traceback\|exception" /tmp/upgrade.log | head -50

# 7. Test on upgraded copy before touching production
```

### Migration Script Location in OpenUpgrade PRs

When contributing to OpenUpgrade:
```
OpenUpgrade/
└── odoo/
    └── addons/
        └── sale/
            └── migrations/
                └── 17.0.1.3/
                    ├── pre-migrate.py
                    └── post-migrate.py
```

---

## Part 2: Enterprise Migration on odoo.sh

### odoo.sh Architecture

```
Production  ←──── promote ──────  Staging
    │                                  │
    │                              Test branch
    │                                  │
    └─────────── branches ─────────────┘
                    │
              Development branch
```

### Upgrade Steps on odoo.sh

**1. Prepare staging branch**
```
odoo.sh dashboard → Branches
→ Create branch from production: "upgrade-v17"
→ This creates a full copy of production DB + filestore
```

**2. Trigger upgrade on staging branch**
```
Branch "upgrade-v17" → Settings → Odoo Version
→ Select "17.0"
→ Click "Upgrade"
→ odoo.sh runs: pg_dump → upgrade scripts → restart
```

**3. Monitor upgrade logs**
```
Branch → Logs tab → "upgrade" step
Look for: ERROR, Traceback, MissingError
```

**4. Test on staging**
- Test all custom modules
- Test all critical workflows
- Verify data integrity
- Check all reports render

**5. Update custom module code**
```bash
# In your branch's repository:
git checkout upgrade-v17
odoo-migrate --directory addons/my_module --init-version-name 16.0 --target-version-name 17.0
git add -A
git commit -m "feat: migrate my_module to v17.0"
git push
# odoo.sh auto-installs/updates on push
```

**6. Write and test migration scripts**
```
addons/my_module/migrations/17.0.x.y.z/
    pre-migrate.py
    post-migrate.py
```
Push to branch → odoo.sh runs scripts automatically on next upgrade run.

**7. Promote to production**
```
Staging branch → "Merge to production"
→ odoo.sh runs upgrade on production DB
→ Zero-downtime mode: runs in maintenance window
```

### odoo.sh Custom Module Constraints

- No shell access to production DB directly
- Use `odoo.sh` Logs for debugging
- `--test-enable` runs on staging automatically
- Filestore is managed by odoo.sh (no manual copy needed)
- SSH access to shell: only for dev/staging branches

```bash
# SSH into odoo.sh branch shell
ssh <branch>@<project>.odoo.com

# Open Odoo shell
~/odoo/odoo-bin shell -d <dbname>

# Check logs
tail -f ~/logs/odoo.log
```

---

## Part 3: On-Premises Migration

### Full On-Prem Upgrade Process

```bash
## ── Step 0: Prerequisites ──────────────────────────────────────────

# Check current version
python odoo-bin --version

# Backup EVERYTHING
pg_dump -Fc mydb > /backup/mydb_v16_$(date +%Y%m%d_%H%M).dump
tar czf /backup/filestore_v16_$(date +%Y%m%d).tar.gz \
    /var/lib/odoo/.local/share/Odoo/filestore/mydb/
# Also backup: custom addons, config files, cron jobs

## ── Step 1: Set up new Odoo version ────────────────────────────────

# Option A: Git (recommended)
git clone https://github.com/odoo/odoo.git -b 17.0 /opt/odoo17
git clone https://github.com/odoo/enterprise.git -b 17.0 /opt/odoo17-enterprise  # EE only

# Option B: For Community via OpenUpgrade
git clone https://github.com/OCA/OpenUpgrade.git -b 17.0 /opt/openupgrade17

# Install Python deps for new version
python3 -m venv /opt/venv-odoo17
source /opt/venv-odoo17/bin/activate
pip install -r /opt/odoo17/requirements.txt
pip install openupgradelib  # for custom migration scripts

## ── Step 2: Migrate custom module code ─────────────────────────────

pip install odoo-migrate
odoo-migrate \
    --directory /opt/custom-addons \
    --init-version-name 16.0 \
    --target-version-name 17.0

## ── Step 3: Create test database ────────────────────────────────────

createdb mydb_v17_test
pg_restore -Fc -d mydb_v17_test /backup/mydb_v16_YYYYMMDD_HHMM.dump
# Copy filestore too
cp -r /var/lib/odoo/.local/share/Odoo/filestore/mydb \
      /var/lib/odoo/.local/share/Odoo/filestore/mydb_v17_test

## ── Step 4: Run upgrade ──────────────────────────────────────────────

# CE: Use OpenUpgrade
python /opt/openupgrade17/odoo-bin \
    -c /etc/odoo17-upgrade.conf \
    -d mydb_v17_test \
    -u all \
    --stop-after-init \
    --logfile=/var/log/odoo/upgrade.log

# EE: Use official Odoo upgrade tool
# Download from: https://upgrade.odoo.com/
python /opt/odoo17/odoo-bin \
    -c /etc/odoo17.conf \
    -d mydb_v17_test \
    -u all \
    --stop-after-init

## ── Step 5: Review logs ──────────────────────────────────────────────

grep -E "ERROR|CRITICAL|Traceback" /var/log/odoo/upgrade.log | less

## ── Step 6: Validate ────────────────────────────────────────────────

# Start new version pointing at test DB
python /opt/odoo17/odoo-bin -c /etc/odoo17.conf -d mydb_v17_test
# Manual testing...

## ── Step 7: Production cutover ──────────────────────────────────────

# Schedule maintenance window
systemctl stop odoo16

# Final backup at cutover
pg_dump -Fc mydb > /backup/mydb_final_before_upgrade.dump

# Run upgrade on production DB
python /opt/openupgrade17/odoo-bin \
    -c /etc/odoo17.conf \
    -d mydb \
    -u all \
    --stop-after-init

# Start v17
systemctl start odoo17
```

### On-Prem Config File for v17

```ini
# /etc/odoo17.conf
[options]
addons_path = /opt/odoo17/addons,/opt/odoo17-enterprise/addons,/opt/custom-addons
data_dir = /var/lib/odoo
db_host = localhost
db_port = 5432
db_user = odoo
db_password = secret
admin_passwd = admin_master_password
xmlrpc_port = 8069
logfile = /var/log/odoo/odoo17.log
log_level = info
workers = 4
max_cron_threads = 2
limit_memory_hard = 2684354560
limit_memory_soft = 2147483648
limit_request = 8192
limit_time_cpu = 600
limit_time_real = 1200
```

---

## Part 4: Odoo Online (SaaS) Migration

### What Odoo Online Is

Odoo Online (odoo.com/start) is a **fully managed SaaS**. You have:
- ✅ Database access via Odoo UI
- ✅ Import/export data via UI
- ✅ Odoo Studio (EE) for no-code customization
- ❌ No shell access
- ❌ No custom Python code / modules
- ❌ No migration scripts
- ❌ No access to PostgreSQL directly

### Upgrade Process (Odoo Online)

```
Settings → General Settings → Database Management
→ "Upgrade" button appears when new version available
→ Odoo triggers automated upgrade
→ Odoo team handles the DB migration
→ You get test URL to validate before it goes live
```

**Your responsibilities before triggering upgrade:**
1. Export all custom data (Studio customizations export if possible)
2. Note all custom reports / email templates
3. Document any third-party app configurations
4. Test on the staging URL Odoo provides

**What Odoo handles automatically:**
- Core module DB migrations
- Filestore migration
- Email template updates
- Report template updates

**What you must manually fix after upgrade:**
- Studio customizations (some may need rebuild)
- Custom email templates broken by field renames
- Scheduled actions referencing removed fields
- Custom Dashboards
- Saved filters referencing removed fields / `attrs`-style domains

### Data Migration from Odoo Online to On-Prem

```bash
# Export from Odoo Online via Settings → Technical → Database → Backup
# Download .zip (includes dump + filestore)

# Restore on-prem
unzip odoo_backup.zip
pg_restore -d mydb odoo_backup/dump.sql
cp -r odoo_backup/filestore/* /var/lib/odoo/.local/share/Odoo/filestore/mydb/
```

---

## Migration Script: Complete Template

```python
# migrations/17.0.1.1.0/pre-migrate.py
"""
Pre-migration: runs BEFORE ORM updates schema.
Use for: column renames, saving data before ORM drops columns.
"""
import logging
from openupgradelib import openupgrade

_logger = logging.getLogger(__name__)


@openupgrade.migrate()
def migrate(env, version):
    if not version:
        _logger.info("Fresh install — skipping pre-migration.")
        return

    _logger.info("Pre-migrating my_module from %s to 17.0.1.1.0", version)
    _rename_fields(env.cr)
    _save_data_before_schema_change(env.cr)


def _rename_fields(cr):
    """Rename DB columns before ORM recreates them."""
    openupgrade.rename_columns(cr, {
        'my_model': [
            ('old_field_name', 'new_field_name'),
            ('type', 'my_type'),  # 'type' is reserved word — rename early
        ],
    })


def _save_data_before_schema_change(cr):
    """Save values from field being removed to temp column."""
    if openupgrade.column_exists(cr, 'my_model', 'legacy_data'):
        openupgrade.logged_query(cr, """
            ALTER TABLE my_model ADD COLUMN IF NOT EXISTS legacy_data_backup TEXT;
            UPDATE my_model SET legacy_data_backup = legacy_data::TEXT;
        """)
```

```python
# migrations/17.0.1.1.0/post-migrate.py
"""
Post-migration: runs AFTER ORM updates schema.
Use for: filling new fields, transforming data, recomputing.
"""
import logging
from openupgradelib import openupgrade

_logger = logging.getLogger(__name__)


@openupgrade.migrate()
def migrate(env, version):
    if not version:
        return

    _logger.info("Post-migrating my_module from %s to 17.0.1.1.0", version)
    _migrate_selection_values(env.cr)
    _fill_new_computed_field(env)
    _migrate_attachments(env)
    _cleanup(env.cr)


def _migrate_selection_values(cr):
    openupgrade.map_values(cr, 'my_model', 'state', [
        ('in_progress', 'confirm'),
        ('validated', 'done'),
    ])


def _fill_new_computed_field(env):
    """Backfill new field from related table data."""
    env.cr.execute("""
        UPDATE my_model m
        SET new_partner_code = p.ref
        FROM res_partner p
        WHERE m.partner_id = p.id
          AND m.new_partner_code IS NULL
    """)


def _migrate_attachments(env):
    """Move binary field to ir.attachment if not already."""
    env.cr.execute("""
        SELECT id, old_binary FROM my_model
        WHERE old_binary IS NOT NULL
    """)
    rows = env.cr.fetchall()
    if not rows:
        return
    for rec_id, data in rows:
        env['ir.attachment'].create({
            'name': f'migrated_{rec_id}.bin',
            'res_model': 'my.model',
            'res_id': rec_id,
            'datas': data,
        })
    openupgrade.logged_query(env.cr, """
        UPDATE my_model SET old_binary = NULL WHERE old_binary IS NOT NULL
    """)


def _cleanup(cr):
    """Remove temp columns added in pre-migrate."""
    openupgrade.drop_columns(cr, [
        ('my_model', 'legacy_data_backup'),
    ])
```

---

## Version-Specific Migration Notes

### v16 → v17 Critical Changes

| Area | What to migrate | Tool |
|------|----------------|------|
| Views: `attrs` | All `attrs=` → inline expr | `odoo-migrate` (review!) |
| Views: `states` | All `states=` → `invisible=` | `odoo-migrate` |
| Views: `<tree>` | → `<list>` | `odoo-migrate` |
| Kanban: `kanban-box` | → `card` template | `odoo-migrate` |
| Python: `name_get()` | → `_compute_display_name()` | `odoo-migrate` |
| Python: `track_visibility` | → `tracking=True` | `odoo-migrate` |
| Python: `@api.model` on create | → `@api.model_create_multi` | `odoo-migrate` |
| JS: `odoo.define()` | → `/** @odoo-module **/` | `odoo-migrate` (partial) |
| JS: Legacy widgets | Complete rewrite in OWL | Manual |
| DB: `ir_ui_view` stored `attrs` | Raw SQL UPDATE | pre-migrate script |
| Sequences: `ir.sequence.date_range` | verify auto-recalc | post-migrate check |

### v17 → v18 Critical Changes

| Area | What to migrate |
|------|----------------|
| `name_get()` remaining calls | Must be gone — removed in v18 |
| `precompute=True` | Add to high-volume create fields |
| `copy()` → `copy_data()` | Refactor copy overrides |
| `<list>` tag | Ensure no `<tree>` in custom views |
| Bootstrap 5 | Check custom CSS (col-xs-* → col-*) |

### v18 → v19 Critical Changes

| Area | What to migrate |
|------|----------------|
| `stored_translations` | Add `translate=True` to stored computed Char fields |
| TypeScript types | Optional: add `.ts` for type safety |
| `row_click_action` | Implement if needed for list UX |

---

## Pre-Upgrade Checklist

```
□ Full DB backup (pg_dump -Fc)
□ Full filestore backup
□ Custom addons listed and versioned
□ Third-party apps (OCA, etc.) have v17/18/19 branches available
□ Run odoo-migrate on all custom modules (review output)
□ Write pre/post migration scripts for schema changes
□ Test upgrade on DB copy (not production)
□ Test all critical business flows manually
□ Test all PDF reports
□ Test all email templates
□ Test scheduled actions
□ Test external API integrations
□ Confirm all OCA modules have OpenUpgrade migration scripts
□ Review upgrade log for ERRORs and fix
□ Performance test on upgraded DB
□ Plan maintenance window for production cutover
□ Communicate downtime to users
```
