---
name: odoo-performance
description: >
  Expert Odoo performance optimization for versions 17.0, 18.0, 19.0 (Community & Enterprise).
  Covers N+1 query detection and fixes, search_fetch, prefetch, ormcache, DB indexes,
  store=True vs False, precompute, bulk create/write, raw SQL, cron chunking, and list view tuning.
  Use when user mentions slow Odoo, N+1 queries, ORM optimization, prefetch, search_fetch,
  ormcache, index, bulk operations, cron timeout, or invokes /odoo-performance.
---

Expert Odoo performance engineer. Target: 17/18/19 CE+EE.

**Always ask what is slow** (page load, compute field, report, import, cron) — the fix depends on the bottleneck.

---

## The Performance Hierarchy

```
1. Fix N+1 queries      → biggest gains, easiest to spot
2. Prefetch / search_fetch → free speed on recordsets
3. DB indexes           → fast search on large tables
4. @ormcache            → eliminate repeated identical queries
5. Optimise compute fields  → store=True vs False tradeoffs
6. Batch large operations   → avoid timeouts and memory blowup
7. Raw SQL for reporting    → bypass ORM for aggregates
8. Server configuration     → workers, memory limits, pooling
```

---

## 1. Finding N+1 Queries

N+1 is the #1 performance killer. It happens when you access a relational field inside a loop.

```bash
# Enable SQL logging — count SELECT statements
./odoo-bin -d yourdb --log-handler=odoo.sql_db:DEBUG 2>&1 | grep "^SELECT" | wc -l
# If count is proportional to record count → N+1
```

```python
# Minimal query counter for shell / tests
class QueryCounter:
    def __init__(self, cr):
        self.cr = cr
        self.count = 0
        self._orig = cr.execute

    def __enter__(self):
        def tracked(q, *a, **kw):
            self.count += 1
            return self._orig(q, *a, **kw)
        self.cr.execute = tracked
        return self

    def __exit__(self, *_):
        self.cr.execute = self._orig

with QueryCounter(self.env.cr) as qc:
    for r in records:
        _ = r.partner_id.name   # N queries!
print(qc.count)
```

### Fix N+1 patterns

```python
# ❌ BAD — 1 query per record
for record in records:
    print(record.partner_id.name)
    print(record.partner_id.country_id.name)   # double N+1

# ✅ GOOD — mapped() prefetches in 1 query
partners = records.mapped('partner_id')              # 1 query
countries = records.mapped('partner_id.country_id')  # 2 queries total

# ✅ GOOD — after mapped(), loop reads from cache (0 extra queries)
for record in records:
    print(record.partner_id.name)

# ✅ GOOD — read() fetches all fields in 1 query
data = records.read(['name', 'partner_id', 'state', 'amount_total'])

# ✅ GOOD — search_read() combines search + read (v17+, source-verified)
data = self.env['your.model'].search_read(
    [('state', '=', 'confirmed')],
    ['name', 'partner_id', 'amount_total'],
    limit=100,
)
```

---

## 2. search_fetch — Prefetch on Search (v17+)

`search_fetch` does a search AND pre-populates the ORM cache for the specified fields in a single round-trip. Source-verified: `odoo/models.py`.

```python
# search_fetch(domain, field_names, offset=0, limit=None, order=None)
records = self.env['your.model'].search_fetch(
    domain=[('state', '=', 'confirmed')],
    field_names=['name', 'partner_id', 'amount_total', 'state'],
    limit=500,
)
# After this: name/partner_id/amount_total/state are cached → 0 extra queries
for rec in records:
    print(rec.partner_id.name, rec.amount_total)  # reads from cache
```

### Prefetch related fields

```python
# Trigger prefetch of partner + country in 2 queries total
_ = records.mapped('partner_id.country_id')
for rec in records:
    print(rec.partner_id.country_id.name)  # free, from cache

# Invalidate prefetch cache (force re-fetch)
records.invalidate_recordset()
records.invalidate_recordset(['partner_id', 'state'])  # specific fields only
```

---

## 3. Optimising Computed Fields

```python
# store=False (default) — no DB column, recomputed on every access
amount_display = fields.Char(compute='_compute_display')

# store=True — in DB, searchable, sortable, 1 recompute per dependency change
amount_total = fields.Float(compute='_compute_total', store=True)

# precompute=True — compute during create(), not in a separate flush (v17+)
# Best for fields needed immediately on record creation
amount_total = fields.Float(
    compute='_compute_total',
    store=True,
    precompute=True,
)

# Decision guide:
# Search/filter/group on it?          → store=True
# Shown in list view for many records? → store=True
# Only shown in form (1 record)?       → store=False
# Very frequently written?             → store=False (avoid write storms)
```

### Narrow depends chains

```python
# ❌ BAD — too broad, recomputes on ANY line change
@api.depends('line_ids')
def _compute_total(self):
    for rec in self:
        rec.amount_total = sum(rec.line_ids.mapped('price_subtotal'))

# ✅ GOOD — only recomputes when price_subtotal changes
@api.depends('line_ids.price_subtotal')
def _compute_total(self):
    for rec in self:
        rec.amount_total = sum(rec.line_ids.mapped('price_subtotal'))
```

### Avoid recompute storms in loops

```python
# ❌ BAD — N recomputes
for line in self.line_ids:
    line.price_subtotal = line.quantity * line.price_unit

# ✅ GOOD — 1 recompute per recordset write
self.line_ids.write({'state': 'confirmed'})
```

---

## 4. Bulk Operations

### Bulk create (always use list form)

```python
# ❌ BAD — N INSERT queries
for item in items:
    self.env['your.model'].create({'name': item['name']})

# ✅ GOOD — single INSERT
self.env['your.model'].create([
    {'name': x['name'], 'partner_id': x['partner_id']}
    for x in items
])

# Very large datasets — chunk to avoid memory issues
CHUNK = 1000
for i in range(0, len(items), CHUNK):
    self.env['your.model'].create([
        {'name': x['name']} for x in items[i:i + CHUNK]
    ])
    self.env.cr.commit()
    self.env.invalidate_all()
```

### Batch write

```python
# ❌ BAD — N UPDATE queries
for record in records:
    record.write({'state': 'confirmed'})

# ✅ GOOD — 1 UPDATE query
records.write({'state': 'confirmed'})

# ✅ GOOD — skip mail tracking overhead on bulk ops
records.with_context(
    mail_notrack=True,
    tracking_disable=True,
    mail_create_nolog=True,
).write({'state': 'confirmed'})
```

### Raw SQL for mass operations

```python
# Bypass ORM entirely for millions of rows
self.env.cr.execute("""
    UPDATE your_model
    SET state = 'confirmed',
        write_date = NOW() AT TIME ZONE 'UTC',
        write_uid = %s
    WHERE state = 'draft'
      AND create_date < NOW() - INTERVAL '30 days'
""", (self.env.uid,))
_logger.info("Auto-confirmed %s records", self.env.cr.rowcount)

# Always invalidate ORM cache after raw SQL
self.env.invalidate_all()
```

---

## 5. Database Indexes

### Add via `init()` method

```python
from odoo import models, tools


class YourModel(models.Model):
    _name = 'your.model'

    state = fields.Selection([...])
    partner_id = fields.Many2one('res.partner')
    date = fields.Date()
    ref = fields.Char(index=True)   # simple btree index via field option

    def init(self):
        # Composite index for common search patterns
        # tools.create_index source-verified in odoo/tools/sql.py v17/v18/v19
        tools.create_index(
            self._cr, 'your_model_state_date_idx',
            self._table, ['state', 'date']
        )
        # Partial index — only index confirmed records (PostgreSQL)
        self._cr.execute("""
            CREATE INDEX IF NOT EXISTS your_model_confirmed_partner_idx
            ON your_model (partner_id)
            WHERE state = 'confirmed'
        """)
```

### When to add an index

```
✅ Add when:
   Field appears in frequent search/filter domains
   Field is used in ORDER BY
   Table has > 10,000 rows and queries are slow

❌ Skip when:
   Table is small (< 1,000 rows)
   Field has very low cardinality (boolean)
   Field is very write-heavy (index slows every INSERT/UPDATE)
```

```sql
-- Inspect existing indexes
SELECT indexname, indexdef
FROM pg_indexes
WHERE tablename = 'your_model'
ORDER BY indexname;

-- Explain query plan
EXPLAIN ANALYZE
SELECT * FROM your_model WHERE state = 'confirmed' AND date > '2024-01-01';
```

---

## 6. @tools.ormcache — Method-Level Caching

Use for expensive lookups that return the same result for the same arguments (config values, reference data, exchange rates).

```python
from odoo import api, models, tools


class YourConfig(models.Model):
    _name = 'your.config'

    @tools.ormcache('config_key')
    def get_value(self, config_key):
        record = self.search([('key', '=', config_key)], limit=1)
        return record.value if record else False

    @tools.ormcache('company_id', 'config_key')
    def get_company_value(self, company_id, config_key):
        record = self.search([
            ('company_id', '=', company_id),
            ('key', '=', config_key),
        ], limit=1)
        return record.value if record else False

    def write(self, vals):
        result = super().write(vals)
        self.clear_caches()   # invalidates all @ormcache on this model
        return result
```

---

## 7. Cron Jobs — Avoid Timeouts

```python
def action_process_large_batch(self):
    """Process up to LIMIT records per cron run — never timeout."""
    import time
    LIMIT = 500
    BUDGET = 300   # 5 minutes
    BUFFER = 50    # stop this many seconds before limit

    start = time.time()
    records = self.search([
        ('state', '=', 'pending'),
        ('scheduled_date', '<=', fields.Datetime.now()),
    ], limit=LIMIT, order='scheduled_date asc')

    processed = 0
    for record in records:
        if time.time() - start > BUDGET - BUFFER:
            _logger.warning(
                "Cron time budget reached at %s/%s", processed, len(records)
            )
            break
        try:
            record._process_single()
            processed += 1
        except Exception as e:
            _logger.error("Failed record %s: %s", record.id, e)
            record.with_context(mail_notrack=True).write({'state': 'error'})
            self.env.cr.rollback()

        if processed % 100 == 0:
            self.env.cr.commit()
            self.env.invalidate_all()

    _logger.info("Cron processed %s/%s in %.1fs", processed, len(records), time.time() - start)
```

---

## 8. List View Performance

```python
# ❌ BAD — store=False computed field in list: 1 compute per row
amount_display = fields.Char(compute='_compute_display')  # store=False

# ✅ GOOD — store=True: list view is a single SELECT
amount_display = fields.Char(compute='_compute_display', store=True)
```

```xml
<!-- Limit default list to recent records -->
<record id="action_your_model" model="ir.actions.act_window">
    <field name="name">Your Models</field>
    <field name="res_model">your.model</field>
    <field name="view_mode">list,form</field>
    <field name="context">{'search_default_my_records': 1}</field>
</record>
```

---

## 9. Quick Wins Checklist

```
□ Replace loop access with records.mapped('field')
□ Replace search()+browse() with search_read()
□ Use search_fetch() for large lists (v17+)
□ store=True on fields shown in list views
□ Batch creates: env['model'].create([...]) not loop
□ Batch writes: records.write({}) not loop
□ mail_notrack=True on bulk writes
□ Add index on frequently filtered non-PK fields
□ @ormcache on config lookups called in loops
□ Chunk cron jobs: 500 records max per run
□ EXPLAIN ANALYZE on queries > 100ms
```

---

## 10. Performance Benchmarking

```python
import time
_logger = logging.getLogger(__name__)


def benchmark(name, func, *args, **kwargs):
    start = time.perf_counter()
    result = func(*args, **kwargs)
    _logger.info("[BENCH] %s: %.3fs", name, time.perf_counter() - start)
    return result

# Compare approaches in shell
t0 = time.perf_counter()
for r in records:
    _ = r.partner_id.name
print(f"Loop: {time.perf_counter()-t0:.3f}s")

t0 = time.perf_counter()
_ = records.mapped('partner_id.name')
print(f"mapped(): {time.perf_counter()-t0:.3f}s")
```

---

## 11. Version Notes

| Feature | v17 | v18 | v19 |
|---------|-----|-----|-----|
| `search_fetch()` | ✅ | ✅ | ✅ |
| `precompute=True` field option | ✅ | ✅ | ✅ |
| `tools.create_index()` | ✅ | ✅ | ✅ |
| `@tools.ormcache` | ✅ | ✅ | ✅ |
| `records.invalidate_recordset()` | ✅ | ✅ | ✅ |
| `env.invalidate_all()` | ✅ | ✅ | ✅ |
