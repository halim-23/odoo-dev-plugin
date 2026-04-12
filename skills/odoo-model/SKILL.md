---
name: odoo-model
description: >
  Odoo model development for versions 17.0, 18.0, 19.0 (Community & Enterprise).
  Full knowledge of all deprecations and changes from v16→v17→v18→v19: name_get removal,
  display_name, _rec_names_search, precompute, copy_data, api.one removal, tracking field,
  model_create_multi, field API changes, ORM best practices, and dynamic domain construction
  (static/string/lambda/computed Char/Binary fields, expression.AND/OR, v19 Domain AST class).
  Use when user asks about Odoo models, fields, ORM, domains, deprecations, or invokes /odoo-model.
---

Expert Odoo model developer. Full deprecation knowledge v16→v17→v18→v19. CE+EE.

## Model Structure (v17+)

```python
from odoo import api, fields, models, _
from odoo.exceptions import UserError, ValidationError

class MyModel(models.Model):
    _name = 'my.model'
    _description = 'My Model'
    _inherit = ['mail.thread', 'mail.activity.mixin']
    _order = 'name asc'
    _rec_name = 'name'
    _check_company_auto = True  # auto-check company_id on write/create (v14+)

    name = fields.Char(required=True, tracking=True)
    active = fields.Boolean(default=True)
    company_id = fields.Many2one('res.company', default=lambda self: self.env.company)
```

---

## Deprecation & Breaking Changes: v16 → v17

### `name_get()` → `_compute_display_name()` ⚠️ DEPRECATED v17, REMOVED v18

```python
# ❌ v16 pattern — DEPRECATED in v17, broken in v18
def name_get(self):
    return [(rec.id, f"[{rec.ref}] {rec.name}") for rec in self]

# ✅ v17+ pattern
def _compute_display_name(self):
    for rec in self:
        rec.display_name = f"[{rec.ref}] {rec.name}"
```

`display_name` is now a real `Char` computed field on `BaseModel`.  
Do NOT call `name_get()` directly — use `rec.display_name` or `rec._origin.display_name`.

### `name_search()` → `_rec_names_search` ⚠️ DEPRECATED v17

```python
# ❌ v16 — override name_search for multi-field search
@api.model
def name_search(self, name='', args=None, operator='ilike', limit=100):
    if name:
        domain = ['|', ('name', operator, name), ('ref', operator, name)]
        return self.search(domain + (args or []), limit=limit).name_get()
    return super().name_search(name, args, operator, limit)

# ✅ v17+ — class attribute, no method override needed
class MyModel(models.Model):
    _rec_names_search = ['name', 'ref', 'partner_id.name']
```

### `track_visibility` → `tracking` ✅ CHANGED v14, REMOVED v19

**Source-verified** (mail_thread.py): `track_visibility` still checked as fallback in v17 AND v18 (`getattr(field, 'tracking', None) or getattr(field, 'track_visibility', None)`). Removed in v19.  
Use `tracking` in all new code regardless — `track_visibility` is never the right choice.

```python
# ❌ OLD (v13 and earlier, removed v19)
name = fields.Char(track_visibility='onchange')
amount = fields.Float(track_visibility='always')

# ✅ v14+ (required v19+)
name = fields.Char(tracking=True)     # track on change
amount = fields.Float(tracking=10)    # tracking=int sets sequence in chatter
```

### `@api.one` ✅ REMOVED v14, causes ImportError v17+

```python
# ❌ REMOVED — raises ImportError in v14+
from odoo import api
@api.one
def my_method(self):
    self.name = 'x'

# ✅ Always iterate manually
def my_method(self):
    for rec in self:
        rec.name = 'x'
```

### `@api.multi` ✅ REMOVED v14, no-op before that

```python
# ❌ REMOVED
@api.multi
def my_method(self):
    ...

# ✅ Just define the method — default behavior is multi-record
def my_method(self):
    for rec in self:
        ...
```

### `@api.model` for `create` → `@api.model_create_multi` ✅ v14+, REQUIRED v17+

```python
# ❌ OLD
@api.model
def create(self, vals):
    return super().create(vals)

# ✅ v14+ handles both single dict and list
@api.model_create_multi
def create(self, vals_list):
    for vals in vals_list:
        if not vals.get('ref'):
            vals['ref'] = self.env['ir.sequence'].next_by_code('my.model') or '/'
    return super().create(vals_list)
```

### `@api.returns` ✅ STILL WORKS but rarely needed

```python
# Only use when method returns recordset and callers depend on that
@api.returns('self', lambda value: value.id)
def copy(self, default=None):
    return super().copy(default)
```

### `fields.Reference` — `selection` must be list of tuples (not callable in v17)

```python
# ❌ callable selection removed for Reference
ref = fields.Reference(selection='_get_ref_models')  # broken v17+

# ✅
ref = fields.Reference(selection=[('res.partner', 'Partner'), ('res.company', 'Company')])
```

### `_inherits` column naming (v17 stricter)

```python
# Delegation inheritance — _inherits key must match existing Many2one field name
class MyExtended(models.Model):
    _name = 'my.extended'
    _inherits = {'res.partner': 'partner_id'}  # partner_id must be defined below
    partner_id = fields.Many2one('res.partner', required=True, ondelete='cascade')
```

---

## Deprecation & Breaking Changes: v17 → v18

### `copy()` → `copy_data()` ✅ NEW v18 preferred

```python
# ❌ Old pattern (still works but not batch-friendly)
def copy(self, default=None):
    default = dict(default or {})
    default['name'] = _('%s (copy)', self.name)
    return super().copy(default)

# ✅ v18+ — copy_data() supports multi-record batch copy
def copy_data(self, default=None):
    vals_list = super().copy_data(default=default)
    for rec, vals in zip(self, vals_list):
        vals['name'] = _('%s (copy)', rec.name)
    return vals_list
```

### `precompute=True` ✅ NEW v18 — store computed fields at create time

```python
# Without precompute: stored field computed lazily after create (2 queries)
# With precompute: value set during create SQL INSERT (1 query, faster bulk)

ref = fields.Char(
    compute='_compute_ref',
    store=True,
    precompute=True,   # <-- compute during create before INSERT
    readonly=True,
)

@api.depends('partner_id.ref', 'date')
def _compute_ref(self):
    for rec in self:
        rec.ref = f"{rec.partner_id.ref or 'XX'}-{rec.date or 'NODATE'}"
```

Use `precompute=True` when:
- Field is always needed immediately after creation
- Creating many records in batch (import, migration)
- Field depends on values set in `vals` at create time

### `display_name` field — source-verified across versions

**Source-verified** (`models.py`):
- **v17**: Already a `Char` field with `compute='_compute_display_name'` (no `store`, no `search`)
- **v18**: Same computed field + added `search='_search_display_name'` method
- **v19**: Promoted to explicit class attribute `display_name = Char(compute=..., search=...)`

The field is **NOT stored** in any version — it is always computed.  
Override `_compute_display_name` only — do NOT add `display_name = fields.Char(...)` to your model.

### `name_get()` ✅ FULLY REMOVED v18

Calling `rec.name_get()` in v18 raises `AttributeError`.  
Use `rec.display_name` or `rec.mapped('display_name')`.

---

## Deprecation & Breaking Changes: v18 → v19

### Translatable computed fields — `stored_translations` (v19)

```python
# v19: computed Char fields can be translated when stored
name_display = fields.Char(
    compute='_compute_name_display',
    store=True,
    translate=True,     # v19: works on stored computed fields
)
```

### `_auto_join` on Many2one (v19 default change)

```python
# v19: Many2one fields searched via JOIN by default for performance
# To disable (rare):
partner_id = fields.Many2one('res.partner', auto_join=False)
```

### `Json` field improvements (v19)

```python
# v19: Json field supports default_factory and schema validation
metadata = fields.Json(
    default=dict,       # callable default
    # schema validation via _check_metadata constrains
)
```

---

## Field Reference (v17+)

### All Field Types

```python
# Text
name = fields.Char(string='Name', size=128, required=True, index=True, translate=True)
description = fields.Text(translate=True)
notes = fields.Html(sanitize=True, strip_style=False)

# Numeric
count = fields.Integer(default=0)
price = fields.Float(digits=(16, 2))
amount = fields.Monetary(currency_field='currency_id')
currency_id = fields.Many2one('res.currency', default=lambda s: s.env.company.currency_id)

# Boolean / State
active = fields.Boolean(default=True)
state = fields.Selection([
    ('draft', 'Draft'),
    ('confirm', 'Confirmed'),
    ('done', 'Done'),
    ('cancel', 'Cancelled'),
], default='draft', tracking=True, index=True)

# Date / Time — always naive UTC datetime, convert with fields.Datetime.context_timestamp()
date = fields.Date(default=fields.Date.today)
datetime_field = fields.Datetime(default=fields.Datetime.now)

# Relational
partner_id = fields.Many2one('res.partner', ondelete='restrict', index=True, check_company=True)
line_ids = fields.One2many('my.model.line', 'order_id', copy=True)
tag_ids = fields.Many2many('my.tag', 'my_model_tag_rel', 'model_id', 'tag_id')

# Binary
document = fields.Binary(attachment=True)  # stored as ir.attachment
image = fields.Image(max_width=1920, max_height=1080)  # auto-resized

# JSON (v16+)
extra_data = fields.Json(default=dict)

# Properties (v16+ EE / v17 CE)
properties = fields.Properties('properties_definition_id', string='Properties')
```

### Computed & Related Fields

```python
# Stored computed (store=True → searchable, groupable)
total = fields.Float(compute='_compute_total', store=True, precompute=True)  # precompute v18+

@api.depends('line_ids.subtotal')
def _compute_total(self):
    for rec in self:
        rec.total = sum(rec.line_ids.mapped('subtotal'))

# Related (shortcut — auto depends, auto store option)
partner_name = fields.Char(related='partner_id.name', store=True, readonly=True)
partner_country_id = fields.Many2one(related='partner_id.country_id', store=True)

# Writable computed
name_upper = fields.Char(compute='_compute_name_upper', inverse='_set_name_upper')

@api.depends('name')
def _compute_name_upper(self):
    for rec in self:
        rec.name_upper = (rec.name or '').upper()

def _set_name_upper(self):
    for rec in self:
        rec.name = (rec.name_upper or '').lower()
```

### Field Parameters Reference

| Parameter | Type | Notes |
|-----------|------|-------|
| `string` | str | label (auto-generated from field name if omitted) |
| `required` | bool | DB NOT NULL + UI validation |
| `readonly` | bool | UI readonly (not DB) |
| `index` | bool/str | DB index; `'btree'`, `'trigram'` (v16+) |
| `store` | bool | persist in DB (default True except computed) |
| `compute` | str | method name |
| `inverse` | str | setter for computed |
| `search` | str | custom search method for non-stored computed |
| `depends` | tuple | `@api.depends` alternative for related |
| `precompute` | bool | compute at create time (v18+) |
| `copy` | bool | include in `copy()` (default True) |
| `default` | val/callable | default value |
| `tracking` | bool/int | mail.thread change tracking |
| `groups` | str | `'module.group_xml_id'` — field hidden from non-members |
| `company_dependent` | bool | value per company (uses `ir.property`) |
| `translate` | bool | field value translatable |
| `check_company` | bool | validate Many2one record belongs to same company |
| `ondelete` | str | `'set null'`, `'restrict'`, `'cascade'` for Many2one |
| `auto_join` | bool | JOIN instead of IN subquery for search (v19 default True) |

---

## ORM Methods

```python
# Search
recs = self.env['my.model'].search([('state', '=', 'draft')], limit=10, order='name asc')

# Search read (single query)
data = self.env['my.model'].search_read([('state', '=', 'draft')], ['name', 'partner_id'], limit=10)

# Count
n = self.env['my.model'].search_count([('state', '!=', 'done')])

# Read (list of dicts)
data = recs.read(['name', 'state', 'partner_id'])

# Create (batch)
records = self.env['my.model'].create([{'name': 'A'}, {'name': 'B'}])

# Write
recs.write({'state': 'done'})

# Unlink
recs.unlink()

# Browse
rec = self.env['my.model'].browse(record_id)  # does NOT hit DB until field access

# Exists check
if not rec.exists():
    raise UserError(_('Record deleted.'))

# Copy
new_rec = rec.copy({'name': f'{rec.name} (copy)'})     # v17
new_recs = recs.copy_data({'name': 'copy'})             # v18+ batch

# mapped / filtered / sorted (in-memory)
names = recs.mapped('name')
partners = recs.mapped('partner_id')                    # returns recordset
draft = recs.filtered(lambda r: r.state == 'draft')
sorted_recs = recs.sorted('name')
sorted_recs = recs.sorted(key=lambda r: r.amount_total, reverse=True)

# grouped (v16+ — group without SQL)
groups = recs.grouped('state')   # dict: {state_value: recordset}

# Environment context
rec.with_context(lang='fr_FR').display_name
rec.with_user(admin).sudo_method()
rec.sudo().write({'state': 'done'})
rec.with_company(company_id).action_confirm()
```

---

## Decorators Reference

| Decorator | Purpose | v17+ notes |
|-----------|---------|------------|
| `@api.depends(*fields)` | trigger compute | use dotted paths: `'line_ids.price'` |
| `@api.depends_context(*keys)` | trigger on context key change | e.g. `'company_id'`, `'lang'` |
| `@api.model` | no self recordset (class method) | only for `search`, `create` helpers |
| `@api.model_create_multi` | batch create (list of dicts) | replaces `@api.model` on `create` |
| `@api.onchange(*fields)` | UI only — triggers in form | return `{'warning': {...}}` or `{}` |
| `@api.constrains(*fields)` | validation, raises `ValidationError` | fires on create/write |
| `@api.ondelete(at_uninstall=False)` | hook before unlink | v14+, replaces unlink override |

---

## Constraints

```python
# Python constraint
@api.constrains('date_start', 'date_end')
def _check_dates(self):
    for rec in self:
        if rec.date_start and rec.date_end and rec.date_start > rec.date_end:
            raise ValidationError(_('End date must be after start date.'))

# SQL constraint (DB-level, faster)
_sql_constraints = [
    ('ref_company_uniq', 'UNIQUE(ref, company_id)', 'Reference must be unique per company.'),
    ('positive_amount', 'CHECK(amount >= 0)', 'Amount cannot be negative.'),
]
```

---

## Inheritance

```python
# Classical (extend existing model, same table)
class ResPartner(models.Model):
    _inherit = 'res.partner'
    custom_code = fields.Char()

# Prototypal (new model extending base, new table)
class SaleOrderCustom(models.Model):
    _name = 'sale.order.custom'
    _inherit = 'sale.order'   # copies all fields+methods to new model

# Delegation (separate table, fields forwarded via JOIN)
class ProjectTaskExtended(models.Model):
    _name = 'project.task.extended'
    _inherits = {'project.task': 'task_id'}
    task_id = fields.Many2one('project.task', required=True, ondelete='cascade')
    custom_field = fields.Char()
```

---

## CRUD Overrides (v17+ patterns)

```python
@api.model_create_multi
def create(self, vals_list):
    for vals in vals_list:
        vals.setdefault('ref', self.env['ir.sequence'].next_by_code('my.model') or '/')
    return super().create(vals_list)

def write(self, vals):
    if 'state' in vals and vals['state'] == 'done':
        for rec in self:
            rec._validate_for_done()
    return super().write(vals)

@api.ondelete(at_uninstall=False)
def _unlink_if_draft(self):
    if any(rec.state != 'draft' for rec in self):
        raise UserError(_('Only draft records can be deleted.'))
# Note: @api.ondelete is v14+ — use unlink() override for compatibility

def unlink(self):
    if any(r.state == 'done' for r in self):
        raise UserError(_('Cannot delete confirmed records.'))
    return super().unlink()

# v18+ batch copy
def copy_data(self, default=None):
    vals_list = super().copy_data(default=default)
    for rec, vals in zip(self, vals_list):
        vals['name'] = _('%s (copy)', rec.name)
    return vals_list
```

---

## Full Deprecation Table

| Feature | Last Good Version | Removed | Replacement |
|---------|------------------|---------|-------------|
| `name_get()` | v17 (deprecated) | v18 | `_compute_display_name()` |
| `name_search()` override | v16 | v17 | `_rec_names_search = [...]` |
| `track_visibility='onchange'` | v13 | v19 | `tracking=True` |
| `track_visibility='always'` | v13 | v19 | `tracking=True` |
| `@api.one` | v13 | v14 | manual `for rec in self` |
| `@api.multi` | v13 | v14 | no decorator (default) |
| `@api.model` on `create` | v13 | v17 warn | `@api.model_create_multi` |
| `fields.Date.today()` → str | v12 | v13 | returns `date` object |
| `fields.Datetime.now()` → str | v12 | v13 | returns `datetime` object |
| `copy()` single-record | v17 | — | `copy_data()` preferred v18+ |
| `fields.Reference` callable selection | v16 | v17 | static list of tuples |
| `_columns` dict | v7 | v10 | `fields.*` class attributes |
| `_defaults` dict | v7 | v10 | `default=` on field |
| `self.pool.get('model')` | v7 | v10 | `self.env['model']` |
| `cr.fetchall()` result as dict | never | — | use `dictfetchall()` helper |
| `context_get()` | v7 | v10 | `self.env.context` |

---

## Dynamic Domain Construction (source-verified v17/v18/v19)

### 1. Static domain on field definition (all versions)

```python
# Literal list — evaluated once at class load
field_id = fields.Many2one('res.partner', domain=[('customer_rank', '>', 0)])

# String with sibling field reference — evaluated at runtime per record
# `company_id` = another field on the SAME model, resolved by the JS client
field_id = fields.Many2one(
    'account.payment.term',
    domain="['|', ('company_id', '=', False), ('company_id', '=', company_id)]"
)
```

String domains use **sibling field names** as variables (not Python evaluation).
`company_id` in the string refers to the `company_id` field on the current record.
Works in v17/v18/v19 — no changes.

### 2. Lambda domain (all versions)

```python
# Lambda called at class setup — `self` is the model class (not a record)
# Use for domains that reference env.ref() or model-level constants
user_id = fields.Many2one(
    'res.users',
    domain=lambda self: "[('groups_id', '=', {}), ('share', '=', False)]".format(
        self.env.ref('sales_team.group_sale_salesman').id
    )
)

# Or pure list — evaluated once, `self` is model class
partner_id = fields.Many2one(
    'res.partner',
    domain=lambda self: [('res_model', '=', self._name)]
)
```

### 3. Computed domain field — `Char` type (all versions, source: stock)

Use when domain depends on **record state** (not just class-level constants):

```python
from ast import literal_eval
import json

class MyWizard(models.TransientModel):
    _name = 'my.wizard'

    company_id = fields.Many2one('res.company')
    location_id = fields.Many2one('stock.location')

    # ① Declare a Char field to hold the domain string
    product_domain = fields.Char(compute='_compute_product_domain')

    # ② Reference it by name in domain= attribute
    product_id = fields.Many2one('product.product', domain="product_domain")

    @api.depends('company_id', 'location_id')
    def _compute_product_domain(self):
        for rec in self:
            domain = [('company_id', '=', rec.company_id.id)]
            if rec.location_id:
                domain += [('location_id', '=', rec.location_id.id)]
            # Char field stores the domain as a string representation
            rec.product_domain = domain  # assign list, ORM serializes it

    def _validate_product(self):
        """Cross-check computed domain against saved Char value."""
        self.ensure_one()
        valid = self.env['product.product'].search_count(
            [('id', '=', self.product_id.id)] + literal_eval(self.product_domain)
        )
        if not valid:
            self.product_id = False
```

In the **XML view**, the computed domain field MUST be declared (hidden) so the client has access to it:

```xml
<form>
    <field name="product_id" domain="product_domain"/>
    <field name="product_domain" invisible="True"/>  <!-- REQUIRED! client needs value -->
</form>
```

In list/tree views use `column_invisible="True"` (not `invisible`):

```xml
<list>
    <field name="product_id" domain="product_domain"/>
    <field name="product_domain" column_invisible="True"/>
</list>
```

### 4. Computed domain field — `Binary` type (source: account, v17/v18/v19 same)

`Binary` field avoids string serialization — stores Python list directly:

```python
class AccountTaxRepartitionLine(models.Model):
    _name = 'account.tax.repartition.line'

    # Binary field: ORM sends raw Python list to client (no string parsing)
    tag_ids_domain = fields.Binary(
        string="tag domain",
        compute="_compute_tag_ids_domain"
    )
    tag_ids = fields.Many2many('account.account.tag', domain="tag_ids_domain")

    @api.depends('company_id', 'country_id')
    def _compute_tag_ids_domain(self):
        for line in self:
            line.tag_ids_domain = [
                ('applicability', '=', 'taxes'),
                ('country_id', 'in', [line.country_id.id, False])
            ]
```

`Binary` vs `Char` for domain fields:

| | `Char` (domain as string) | `Binary` (domain as list) |
|---|---|---|
| Assignment | assign list, ORM may serialize | assign list directly |
| Source example | `stock.quant.relocate` | `account.tax.repartition.line` |
| Read-back | `literal_eval(rec.field)` | use as-is |
| Client handling | parsed by JS client | sent as JSON |
| Versions | v17/v18/v19 | v17/v18/v19 |

### 5. `odoo.osv.expression` helpers — AND/OR/normalize (all versions)

```python
from odoo.osv import expression

base_domain = [('active', '=', True)]
company_domain = [('company_id', '=', self.company_id.id)]
extra_domain = [('state', 'in', ['draft', 'sale'])]

# AND: all conditions must match
combined = expression.AND([base_domain, company_domain, extra_domain])

# OR: any condition must match
combined = expression.OR([base_domain, company_domain])

# normalize: convert [('f', 'op', 'v')] → ['&', ('f', 'op', 'v'), ...]
# (adds implicit & prefix, required for raw domains passed to search)
normalized = expression.normalize_domain(base_domain)

# Typical pattern: merge user-provided domain with security domain
def _get_records(self, domain=None):
    base = [('company_id', 'in', self.env.company.ids)]
    return self.search(expression.AND([base, domain or []]))
```

`expression.AND/OR` available in v17/v18/v19 — no changes to signatures.

### 6. NEW in v19: `Domain` class — AST-based domain (v19 ONLY)

```python
from odoo.fields import Domain  # re-exported from odoo.orm.domains

# Build from legacy list (backward compatible)
d = Domain([('active', '=', True), ('company_id', '=', 5)])

# Build a single condition
d = Domain('active', '=', True)

# Combine with operators (& | ~)
d1 = Domain('active', '=', True)
d2 = Domain('company_id', '=', 5)
d3 = Domain('state', 'in', ['draft', 'sale'])

combined = d1 & d2          # AND
combined = d1 | d2          # OR
inverted = ~d1              # NOT

# Class-level combinators
Domain.AND([d1, d2, d3])    # n-ary AND
Domain.OR([d1, d2, d3])     # n-ary OR

# Constants
Domain.TRUE     # equivalent to []
Domain.FALSE    # equivalent to [('id', '=', False)]

# Domain is IMMUTABLE (v19) — no setattr
# Iterating over Domain gives legacy list representation (backward compat)
list(Domain('active', '=', True))  # → [('active', '=', True)]
domain_list = list(d1 & d2)        # → ['&', ('active', '=', True), ('company_id', '=', 5)]
```

`odoo.osv.expression.AND/OR/normalize_domain` still work in v19 for backward compatibility.

| API | v17 | v18 | v19 |
|-----|-----|-----|-----|
| `expression.AND(list)` | ✅ | ✅ | ✅ |
| `expression.OR(list)` | ✅ | ✅ | ✅ |
| `expression.normalize_domain()` | ✅ | ✅ | ✅ |
| `Domain` class from `odoo.fields` | ❌ | ❌ | ✅ NEW |
| `Domain & Domain` / `Domain \| Domain` | ❌ | ❌ | ✅ NEW |
| `Domain.AND([...])` / `Domain.OR([...])` | ❌ | ❌ | ✅ NEW |
| `Domain.TRUE` / `Domain.FALSE` | ❌ | ❌ | ✅ NEW |

---

## Best Practices (v17+)

- Iterate `for rec in self` always — never assume singleton unless `ensure_one()`
- Use `_('...')` for all user-facing strings; `%(var)s` not `%s` in translatable strings
- `ondelete='restrict'` default for business Many2one; `cascade` only for detail lines
- Never `search()` inside loop — batch with `mapped()` or pre-load
- `store=True` on computed only if search/group-by needed; adds DB maintenance cost
- `index=True` on Many2one, frequently filtered Char/Integer
- `index='trigram'` for fuzzy text search (PostgreSQL pg_trgm)
- `check_company=True` on cross-company Many2one fields
- `precompute=True` (v18+) on stored computed fields needed at create time
- `sudo()` only when intentional — always comment why
