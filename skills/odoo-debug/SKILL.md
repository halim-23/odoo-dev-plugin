---
name: odoo-debug
description: >
  Odoo debugging and troubleshooting for versions 17.0, 18.0, 19.0 (Community & Enterprise).
  Covers debug mode, logging, ORM query analysis, shell, common errors and fixes,
  performance profiling, and developer tools. Use when user asks about debugging Odoo,
  errors, performance, logs, shell, or invokes /odoo-debug.
---

Expert Odoo debugger. Target: 17/18/19 CE+EE.

## Enable Debug Mode

```
# URL parameter
https://myodoo.com/web?debug=1          # debug mode
https://myodoo.com/web?debug=assets     # also disables asset bundling
https://myodoo.com/web?debug=0          # disable

# Settings → General Settings → Developer Tools → Activate the developer mode
```

## Logging Configuration

```python
# odoo.conf
log_level = debug         # debug, info, warning, error, critical
log_handler = :DEBUG      # all loggers at DEBUG

# Per-module logging in conf:
log_handler = odoo.addons.my_module:DEBUG,odoo.sql_db:DEBUG

# Or at runtime (restart not needed with RPC):
import logging
_logger = logging.getLogger(__name__)

_logger.debug("Debug: %s", variable)
_logger.info("Info: %s", value)
_logger.warning("Warning")
_logger.error("Error: %s", e, exc_info=True)
```

## Odoo Shell

```bash
# Start shell for database mydb
python odoo-bin shell -d mydb --no-http

# Useful shell commands
env['my.model'].search([])
env['my.model'].browse(1).read()
env['my.model'].browse(1).name_get()

# Check field value
rec = env['my.model'].browse(5)
print(rec.name, rec.state, rec.partner_id.name)

# Execute method
rec.action_confirm()
env.cr.commit()  # commit if you want to persist

# Rollback if testing
env.cr.rollback()

# Run as different user
env2 = env(user=env.ref('base.user_demo'))
env2['my.model'].search([])
```

## SQL Query Logging

```python
# In odoo.conf:
log_handler = odoo.sql_db:DEBUG

# Or per-session in code:
import logging
logging.getLogger('odoo.sql_db').setLevel(logging.DEBUG)

# Count queries in a block
from odoo.tools import sql
# Use profiler (v15+)
with env.cr.savepoint():
    from odoo.tools.profiler import Profiler
    with Profiler():
        records = env['my.model'].search([])
```

## Common Errors & Fixes

### `MissingError: Record does not exist`
```python
# Wrong: browsing deleted record
rec = env['my.model'].browse(99999)
rec.name  # raises MissingError

# Fix: check exists
rec = env['my.model'].browse(99999)
if rec.exists():
    print(rec.name)
```

### `psycopg2.IntegrityError: FOREIGN KEY violation`
- Check `ondelete=` on Many2one fields
- Ensure `ir.model.access.csv` allows unlink
- Check `_sql_constraints` unique violations

### `UserError: You cannot delete a record` 
- Model has `unlink()` override with check — read error message
- Record in "done" or locked state

### `AttributeError: 'my.model' object has no attribute 'xyz'`
- Field not defined in model
- Module not updated: `python odoo-bin -d mydb -u my_module`

### `ValueError: External ID not found`
- Missing `ref="..."` in XML data
- Module not installed — check `depends` in manifest

### `ir.rule` hiding records unexpectedly
```python
# Debug: check which rules apply
env['ir.rule'].sudo()._get_domain(env['my.model'], 'read')
# Or bypass rules temporarily with sudo
records = env['my.model'].sudo().search([])
```

### Computed field not updating
```python
# Force recompute
rec.modified(['dependency_field'])
rec._recompute_recordset()
# Or: mark for recompute
env.add_to_compute(rec._fields['computed_field'], rec)
```

### `Translation not found` / strings not translating
```bash
# Regenerate .po file
python odoo-bin --i18n-export=/tmp/my_module.pot -d mydb -l en_US --modules=my_module
# Import translations
python odoo-bin --i18n-import=/tmp/my_module_fr.po -d mydb -l fr_FR --modules=my_module
```

## Performance Debugging

### N+1 Query Detection

```python
# BAD: N+1
for rec in records:
    print(rec.partner_id.name)  # query per record

# GOOD: prefetch (Odoo does this automatically with browse, but explicit for clarity)
records.mapped('partner_id.name')  # single query
```

### Disable Prefetch (for memory)

```python
env['my.model'].with_prefetch().search([])  # custom prefetch set
env['my.model'].with_context(prefetch_fields=False).search([])
```

### Analyze Slow Queries

```sql
-- In psql
EXPLAIN ANALYZE
SELECT * FROM my_model WHERE state = 'draft' ORDER BY name;

-- Check missing indexes
SELECT schemaname, tablename, attname, n_distinct, correlation
FROM pg_stats
WHERE tablename = 'my_model';
```

Add index in Python:
```python
partner_id = fields.Many2one('res.partner', index=True)
```

### Profiling (v15+)

```python
# Enable profiler from Settings → Technical → Profiler
# Or programmatically:
from odoo.tools.profiler import Profiler, profiler

with Profiler(db=env.cr.dbname):
    slow_method()
```

## Upgrade / Update Module

```bash
# Update specific module
python odoo-bin -d mydb -u my_module

# Update with dependencies
python odoo-bin -d mydb -u my_module,dep_module

# Run tests during update
python odoo-bin -d mydb -u my_module --test-enable --test-tags my_module

# Full upgrade (slow)
python odoo-bin -d mydb -u all
```

## Browser Dev Tools (OWL)

```javascript
// In browser console:
// Get OWL component from DOM element
const el = document.querySelector('.o_my_component');
const component = owl.__apps__[0].getNode(el)?.component;

// Access env services
odoo.__DEBUG__.services['orm']
odoo.__DEBUG__.services['notification']
```

## Version-Specific Debug

**v17:** Debug mode toggle in URL: `?debug=assets` disables bundle minification.
**v17:** `?debug=tests` runs browser QUnit tests.
**v18:** Built-in query profiler in Settings when debug mode active.
**v19:** `ODOO_DISABLE_ASSETS_CACHE=1` env var forces asset recompilation.

## Useful One-liners (Shell)

```python
# Find all records in error state
env['my.model'].search([('state', '=', 'error')]).mapped('name')

# Check user permissions
user = env.ref('base.user_demo')
user.has_group('my_module.group_my_module_manager')

# Inspect field definitions
env['my.model']._fields['state']
env['my.model']._fields['state'].selection

# Find XML ID of a record
env['ir.model.data'].search([('model', '=', 'res.partner'), ('res_id', '=', 1)])

# Recompute all stored fields of model
env['my.model']._recompute_all()  # v17+

# Flush pending writes
env.flush_all()
```
