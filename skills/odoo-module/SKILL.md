---
name: odoo-module
description: >
  Odoo module scaffolding and structure for versions 17.0, 18.0, 19.0 (Community & Enterprise).
  Covers __manifest__.py, __init__.py, directory layout, data loading order, assets,
  demo data, module dependencies, and ir.cron scheduled actions (field changes v17→v18→v19,
  numbercall/doall removed, failure tracking, CompletionStatus, progress model).
  Use when user asks about creating a new Odoo module, addon structure, scheduled actions,
  ir.cron, or invokes /odoo-module.
---

Expert Odoo module architect. Target: 17/18/19 CE+EE.

## Directory Layout

```
my_module/
├── __init__.py
├── __manifest__.py
├── models/
│   ├── __init__.py
│   └── my_model.py
├── views/
│   ├── my_model_views.xml
│   └── menus.xml
├── security/
│   ├── ir.model.access.csv
│   └── security.xml          # record rules / groups
├── data/
│   └── sequence_data.xml
├── demo/
│   └── demo_data.xml
├── wizards/
│   ├── __init__.py
│   └── my_wizard.py
├── controllers/
│   ├── __init__.py
│   └── main.py
├── reports/
│   ├── my_report.xml
│   └── my_report_template.xml
├── static/
│   ├── description/
│   │   ├── icon.png          # 128x128
│   │   └── index.html        # optional app description
│   └── src/
│       ├── js/
│       ├── css/
│       └── xml/              # OWL templates
└── tests/
    ├── __init__.py
    └── test_my_model.py
```

## __manifest__.py

```python
{
    'name': 'My Module',
    'version': '17.0.1.0.0',  # odoo_ver.major.minor.patch.hotfix
    'summary': 'Short one-line description',
    'description': """
Long description in RST format.
Multi-line supported.
    """,
    'author': 'My Company',
    'website': 'https://mycompany.com',
    'license': 'LGPL-3',      # or OPL-1 for proprietary
    'category': 'Accounting/Accounting',  # use Odoo category path
    'depends': [
        'base',
        'mail',
        # 'account', 'sale', 'purchase', etc.
    ],
    'data': [
        # Load order matters — security first, then data, then views
        'security/ir.model.access.csv',
        'security/security.xml',
        'data/sequence_data.xml',
        'views/my_model_views.xml',
        'views/menus.xml',
        'reports/my_report.xml',
        'reports/my_report_template.xml',
    ],
    'demo': [
        'demo/demo_data.xml',
    ],
    'assets': {
        'web.assets_backend': [
            'my_module/static/src/js/my_component.js',
            'my_module/static/src/css/my_style.css',
            'my_module/static/src/xml/my_component.xml',
        ],
        'web.assets_frontend': [
            'my_module/static/src/js/website_component.js',
        ],
    },
    'installable': True,
    'application': False,      # True only for top-level app
    'auto_install': False,
}
```

## __init__.py (root)

```python
from . import models
from . import controllers  # if exists
from . import wizards      # if exists
```

## models/__init__.py

```python
from . import my_model
from . import res_partner  # inherited models last
```

## Sequence Data

```xml
<odoo>
    <data noupdate="1">
        <record id="sequence_my_model" model="ir.sequence">
            <field name="name">My Model Sequence</field>
            <field name="code">my.model</field>
            <field name="prefix">MM/%(year)s/</field>
            <field name="padding">5</field>
            <field name="company_id" eval="False"/>
        </record>
    </data>
</odoo>
```

## Demo Data

```xml
<odoo>
    <data>
        <record id="my_model_demo_1" model="my.model">
            <field name="name">Demo Record 1</field>
            <field name="partner_id" ref="base.res_partner_1"/>
            <field name="state">draft</field>
        </record>
    </data>
</odoo>
```

## Version Notes

**v17:** `version` field in manifest supports 5-part (`17.0.x.y.z`).
**v17:** `assets` dict replaces old `qweb` list for templates.
**v17:** `license` field required — omit causes install warning.
**v18:** `'external_dependencies': {'python': ['pandas'], 'bin': ['wkhtmltopdf']}` supported.
**v19:** `'maintainer'` field added to manifest spec.

## Common depends

| Feature | depends |
|---------|---------|
| Chatter | `mail` |
| Partners | `base` (included) |
| Sales | `sale_management` |
| Purchase | `purchase` |
| Inventory | `stock` |
| Accounting | `account` |
| Project | `project` |
| HR | `hr` |
| Website | `website` |
| Portal | `portal` |
| Enterprise spreadsheet | `spreadsheet_edition` |

## Tests

```python
from odoo.tests import tagged
from odoo.tests.common import TransactionCase

@tagged('post_install', '-at_install')
class TestMyModel(TransactionCase):

    def setUp(self):
        super().setUp()
        self.my_model = self.env['my.model'].create({'name': 'Test'})

    def test_initial_state(self):
        self.assertEqual(self.my_model.state, 'draft')

    def test_confirm(self):
        self.my_model.action_confirm()
        self.assertEqual(self.my_model.state, 'confirm')
```

Run: `python odoo-bin -d mydb --test-enable --test-tags my_module -i my_module`

---

## Scheduled Actions (`ir.cron`) — Version Changes (source-verified)

### Fields matrix — v17 vs v18 vs v19

| Field | v17 | v18 | v19 | Notes |
|-------|-----|-----|-----|-------|
| `cron_name` | ✅ | ✅ | ✅ | computed from `ir_actions_server_id.name` |
| `user_id` | ✅ | ✅ | ✅ | runs with this user's context |
| `active` | ✅ | ✅ | ✅ | |
| `interval_number` | ✅ | ✅ required, `aggregator=None` | ✅ required, `aggregator='avg'` | |
| `interval_type` | ✅ | ✅ | ✅ | minutes/hours/days/weeks/months |
| `nextcall` | ✅ | ✅ | ✅ | next scheduled execution |
| `lastcall` | ✅ | ✅ | ✅ | set automatically on run |
| `priority` | ✅ | ✅ `aggregator=None` | ✅ | 0=highest, 10=lowest; default=5 |
| `numbercall` | ✅ | ❌ **REMOVED** | ❌ | `-1` = unlimited was common; now always unlimited |
| `doall` | ✅ | ❌ **REMOVED** | ❌ | "Repeat Missed" flag; now always False behavior |
| `failure_count` | ❌ | ✅ NEW | ✅ | consecutive failure counter; reset on success |
| `first_failure_date` | ❌ | ✅ NEW | ✅ | date of first failure in current streak |

SQL constraint added in v18/v19: `CHECK(interval_number > 0)` — zero/negative interval raises error.

### XML data record — v17 vs v18/v19

```xml
<!-- ✅ v17 pattern -->
<record id="my_cron_job" model="ir.cron">
    <field name="name">My Module: Process Records</field>
    <field name="model_id" ref="my_module.model_my_model"/>
    <field name="state">code</field>
    <field name="code">model._cron_process_records()</field>
    <field name="interval_number">1</field>
    <field name="interval_type">hours</field>
    <field name="numbercall">-1</field>   <!-- -1 = unlimited, required in v17 -->
    <field name="doall" eval="False"/>
    <field name="active" eval="True"/>
    <field name="priority">5</field>
</record>

<!-- ✅ v18/v19 pattern — numbercall and doall REMOVED -->
<record id="my_cron_job" model="ir.cron">
    <field name="name">My Module: Process Records</field>
    <field name="model_id" ref="my_module.model_my_model"/>
    <field name="state">code</field>
    <field name="code">model._cron_process_records()</field>
    <field name="interval_number">1</field>
    <field name="interval_type">hours</field>
    <field name="active" eval="True"/>
    <field name="priority">5</field>
</record>
```

`state="code"` + `code="model._method()"` is the only supported pattern in all versions.  
`model` in the `code` field refers to the model specified by `model_id`.

### Python cron method pattern

```python
class MyModel(models.Model):
    _name = 'my.model'

    def _cron_process_records(self):
        """Called by ir.cron scheduler. Use self.env, no args."""
        records = self.search([('state', '=', 'pending')])
        for record in records:
            record._do_something()
```

Access `lastcall` via context (same all versions):
```python
def _cron_process_records(self):
    lastcall = self.env.context.get('lastcall')  # datetime or False
    domain = [('write_date', '>=', lastcall)] if lastcall else []
    records = self.search(domain)
```

### Triggering a cron manually from Python — `_trigger()`

```python
# Trigger asap (all versions — same signature)
cron = self.env.ref('my_module.my_cron_job')
cron._trigger()

# Trigger at specific datetime
from datetime import datetime, timedelta
cron._trigger(at=datetime.now() + timedelta(hours=2))

# Trigger at multiple times (v17+)
cron._trigger(at=[datetime1, datetime2])
```

Source: `_trigger()` signature same in v17/v18/v19 — accepts `datetime | list[datetime] | None`.

### Failure handling — NEW in v18, extended in v19

**v18+** added automatic failure tracking and admin notification:

```python
# v18/v19: cron failure behavior
# 1. Job raises exception → failure_count += 1, first_failure_date set
# 2. After enough failures → cron auto-deactivated + admin email sent

# Constants (from ir_cron.py):
MAX_FAIL_TIME = timedelta(hours=5)                   # v17/v18/v19
CONSECUTIVE_TIMEOUT_FOR_FAILURE = 3                  # v18/v19
MIN_FAILURE_COUNT_BEFORE_DEACTIVATION = 5            # v19
MIN_DELTA_BEFORE_DEACTIVATION = timedelta(days=7)    # v19 (both thresholds needed)
```

v17: failure just logs an error, no tracking.  
v18: `failure_count` increments; admin notified via `_notify_admin()`; cron deactivated after threshold.  
v19: both `MIN_FAILURE_COUNT_BEFORE_DEACTIVATION` AND `MIN_DELTA_BEFORE_DEACTIVATION` must be exceeded before deactivation (less aggressive).

### `CompletionStatus` — v19 only

v19 introduced `CompletionStatus` class to track job outcome:

```python
# v19 internal values (used in _run_job return)
CompletionStatus.FULLY_DONE     # job completed all work
CompletionStatus.PARTIALLY_DONE # job did some work (batch), more remains
CompletionStatus.FAILED         # job raised exception
```

Cron is rescheduled immediately (`_reschedule_asap`) if `PARTIALLY_DONE`, later if `FULLY_DONE`.

### Progress tracking — `ir.cron.progress`

**v18** introduced `ir.cron.progress` model for long-running crons:

```python
# v18: _notify_progress() — report progress
def _cron_big_job(self):
    records = self.search([])
    total = len(records)
    for i, record in enumerate(records):
        record._process()
        if i % 100 == 0:
            self._notify_progress(done=i, remaining=total - i)
```

**v19**: `_notify_progress()` DEPRECATED → use `_commit_progress()`:

```python
# v19: _commit_progress() — recommended replacement
def _cron_big_job(self):
    records = self.search([])
    for record in records:
        record._process()
        self.env['ir.cron'].browse(
            self.env.context.get('ir_cron_progress_id')
        )._commit_progress(done=1)
```

`ir.cron.progress` fields:

| Field | v18 | v19 |
|-------|-----|-----|
| `cron_id` | ✅ | ✅ |
| `remaining` | ✅ | ✅ |
| `done` | ❌ | ✅ NEW |
| `deactivate` | ❌ | ✅ NEW |
| `timed_out_counter` | ❌ | ✅ NEW |

### `unlink()` → `_unlink_unless_running()` — v19

```python
# v17/v18: standard unlink (raises UserError if cron is running via _try_lock)
cron.unlink()

# v19: replaced with @api.ondelete decorator — same error, different pattern
# Source: @api.ondelete(at_uninstall=False)
# def _unlink_unless_running(self):
#     try:
#         self.lock_for_update()
#     except LockError:
#         raise UserError("...currently being executed...")

# v19 also added `LockError` to imports from odoo.exceptions
```

### `_acquire_one_job()` signature changes

```python
# v17: takes list of job_ids
_acquire_one_job(cls, cr, job_ids)   # ← list

# v18: takes single job_id
_acquire_one_job(cls, cr, job_id)    # ← single int

# v19: type hints + include_not_ready kwarg
_acquire_one_job(cr, job_id: int, *, include_not_ready: bool = False) -> dict | None
```

### Loading cron data in module

Always in `data/` (not `demo/`), wrapped in `<data noupdate="1">`:

```xml
<!-- my_module/data/ir_cron_data.xml -->
<odoo>
    <data noupdate="1">
        <record id="my_cron_job" model="ir.cron">
            ...
        </record>
    </data>
</odoo>
```

In `__manifest__.py`:
```python
'data': [
    'security/ir.model.access.csv',
    'data/ir_cron_data.xml',   # load after models
],
```
