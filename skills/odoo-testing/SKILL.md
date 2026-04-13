---
name: odoo-testing
description: >
  Expert Odoo automated testing for versions 17.0, 18.0, 19.0 (Community & Enterprise).
  Covers TransactionCase, HttpCase, Form helper, mock/patch, tour tests (JS), test tags,
  assertRecordValues, setUp/setUpClass, access rights tests, and running tests with --test-enable.
  Use when user mentions writing tests, TransactionCase, HttpCase, test tags, assertRecordValues,
  mock, patch, tour test, browser_js, or invokes /odoo-testing.
---

Expert Odoo test engineer. Target: 17/18/19 CE+EE.

**Always confirm** Odoo version and what needs testing (Python logic, view/UI, workflow, HTTP endpoint).

---

## Test Types Overview

```
TransactionCase   → Standard unit/integration tests — each test runs in a rolled-back transaction
HttpCase          → Full HTTP + browser tests — tests request/response and UI tours
tagged()          → Decorator to mark tests for selective --test-tags execution
Form              → Simulates a user filling a form view, fires all @api.onchange
```

> **Note:** `SavepointCase` was removed — use `TransactionCase` for all Python tests in v17/v18/v19.

---

## 1. File Structure

```
your_module/
└── tests/
    ├── __init__.py
    ├── test_your_model.py
    ├── test_wizard.py
    ├── test_access.py
    └── test_tours.py
```

```python
# tests/__init__.py
from . import test_your_model
from . import test_wizard
from . import test_access
from . import test_tours
```

---

## 2. TransactionCase — Standard Test Class

Every test method runs in a transaction rolled back after the test. No data leaks between tests.

```python
from odoo.tests.common import TransactionCase, Form
from odoo.exceptions import UserError, ValidationError, AccessError
from odoo.tests import tagged


@tagged('post_install', '-at_install')   # run after install, not during
class TestYourModel(TransactionCase):

    @classmethod
    def setUpClass(cls):
        """Runs ONCE before all tests — use for shared fixtures."""
        super().setUpClass()

        # Disable mail overhead for speed
        cls.env = cls.env(context={
            'mail_create_nolog': True,
            'mail_notrack': True,
            'tracking_disable': True,
            'no_reset_password': True,
        })

        cls.partner = cls.env.ref('base.res_partner_1')
        cls.company = cls.env.ref('base.main_company')

        cls.test_user = cls.env['res.users'].create({
            'name': 'Test User',
            'login': 'testuser_unique@test.local',
            'groups_id': [(4, cls.env.ref('base.group_user').id)],
        })

    def setUp(self):
        """Runs before EACH test — create per-test records here."""
        super().setUp()
        self.record = self.env['your.model'].create({
            'name': 'Test Record',
            'partner_id': self.partner.id,
        })

    # ── CRUD ────────────────────────────────────────────────────

    def test_create_defaults(self):
        record = self.env['your.model'].create({
            'name': 'New',
            'partner_id': self.partner.id,
        })
        self.assertEqual(record.state, 'draft')
        self.assertTrue(record.id)

    def test_compute_field(self):
        self.record.write({'quantity': 5, 'price_unit': 10.0})
        self.assertEqual(self.record.price_subtotal, 50.0)

    # ── State transitions ────────────────────────────────────────

    def test_confirm_from_draft(self):
        self.record.action_confirm()
        self.assertEqual(self.record.state, 'confirmed')

    def test_cannot_confirm_twice(self):
        self.record.action_confirm()
        with self.assertRaises(UserError):
            self.record.action_confirm()

    # ── assertRecordValues — multi-field check in one call ───────

    def test_confirmed_values(self):
        self.record.action_confirm()
        self.assertRecordValues(self.record, [{
            'state': 'confirmed',
            'partner_id': self.partner.id,
            'name': 'Test Record',
        }])

    def test_multiple_records(self):
        rec2 = self.env['your.model'].create({'name': 'Rec 2', 'partner_id': self.partner.id})
        self.assertRecordValues(self.record | rec2, [
            {'name': 'Test Record', 'state': 'draft'},
            {'name': 'Rec 2',       'state': 'draft'},
        ])

    # ── Constraint tests ────────────────────────────────────────

    def test_negative_amount_raises(self):
        with self.assertRaises(ValidationError):
            self.record.write({'amount': -100.0})

    def test_unique_name_constraint(self):
        from psycopg2 import IntegrityError
        with self.assertRaises((IntegrityError, ValidationError)):
            self.env['your.model'].create({
                'name': self.record.name,
                'partner_id': self.partner.id,
            })

    # ── Access / security ───────────────────────────────────────

    def test_user_cannot_delete(self):
        with self.assertRaises(AccessError):
            self.record.with_user(self.test_user).unlink()

    def test_manager_can_delete(self):
        manager = self.env['res.users'].create({
            'name': 'Test Manager',
            'login': 'manager_unique@test.local',
            'groups_id': [(4, self.env.ref('your_module.group_your_module_manager').id)],
        })
        self.record.with_user(manager).unlink()   # should NOT raise

    # ── Exception message check ──────────────────────────────────

    def test_error_message_content(self):
        self.record.action_confirm()
        with self.assertRaises(UserError) as cm:
            self.record.action_confirm()
        self.assertIn("Only draft", str(cm.exception))
```

---

## 3. Form Helper — Simulating UI Onchanges

`Form` simulates a user filling a form view. All `@api.onchange` methods fire automatically.

```python
from odoo.tests.common import Form, TransactionCase


class TestFormOnchange(TransactionCase):

    def test_onchange_fires(self):
        with Form(self.env['your.model']) as form:
            form.partner_id = self.partner       # triggers @api.onchange('partner_id')
            form.quantity = 3
            # Check onchange result before saving
            self.assertEqual(form.currency_id, self.env.company.currency_id)
            record = form.save()

        self.assertEqual(record.quantity, 3)

    def test_edit_existing(self):
        with Form(self.record) as form:
            form.name = 'Updated'
        self.assertEqual(self.record.name, 'Updated')

    def test_one2many_lines(self):
        with Form(self.record) as form:
            with form.line_ids.new() as line:
                line.product_id = self.env.ref('product.product_product_1')
                line.quantity = 2
                line.price_unit = 50.0
            self.assertAlmostEqual(form.amount_total, 100.0)
```

---

## 4. Mocking & Patching

```python
from unittest.mock import patch, MagicMock
from odoo.tests.common import TransactionCase


class TestWithMocks(TransactionCase):

    def test_mock_external_api(self):
        with patch('your_module.models.your_model.requests.post') as mock_post:
            mock_post.return_value.status_code = 200
            mock_post.return_value.json.return_value = {'id': '123', 'status': 'ok'}
            result = self.record.send_to_external_api()
            mock_post.assert_called_once()
            self.assertEqual(result['status'], 'ok')

    def test_mock_email_send(self):
        with patch.object(type(self.env['mail.mail']), 'send', return_value=None) as mock_send:
            self.record.action_send_notification()
            self.assertTrue(mock_send.called)

    def test_fixed_date(self):
        import datetime
        fixed = datetime.date(2024, 1, 15)
        with patch('odoo.fields.Date.today', return_value=fixed):
            self.record.action_set_today()
            self.assertEqual(self.record.date, fixed)

    def test_with_other_company(self):
        other_co = self.env['res.company'].create({'name': 'Test Co'})
        rec = self.record.with_company(other_co)
        self.assertEqual(rec.env.company, other_co)
```

---

## 5. Running Tests

```bash
# Run all tests for a module
./odoo-bin -d test_db -i your_module --test-enable --stop-after-init

# Run with test tags
./odoo-bin -d test_db --test-enable --test-tags your_module --stop-after-init

# Run a specific class
./odoo-bin -d test_db --test-enable \
    --test-tags your_module.TestYourModel --stop-after-init

# Run a specific test method
./odoo-bin -d test_db --test-enable \
    --test-tags your_module.TestYourModel.test_confirm_from_draft --stop-after-init

# Verbose output
./odoo-bin -d test_db --test-enable --log-level=test \
    --test-tags your_module --stop-after-init

# Multiple modules at once
./odoo-bin -d test_db --test-enable --test-tags module_a,module_b --stop-after-init
```

### Test tag rules
```python
from odoo.tests import tagged

@tagged('post_install', '-at_install')   # runs after module install — most common
@tagged('at_install')                     # runs during install only
@tagged('-standard', 'slow')             # exclude from standard run, add custom tag
@tagged('post_install', '-at_install', 'my_custom_tag')
```

```bash
# Use custom tag
--test-tags my_custom_tag

# Exclude a tag
--test-tags -slow

# Combine: post_install but not slow
--test-tags post_install,-slow
```

---

## 6. HttpCase — HTTP & Browser Tests

```python
from odoo.tests.common import HttpCase
from odoo.tests import tagged


@tagged('post_install', '-at_install')
class TestYourHttp(HttpCase):

    def test_portal_page(self):
        self.authenticate('portal', 'portal')
        response = self.url_open('/my/your-model')
        self.assertEqual(response.status_code, 200)
        self.assertIn(b'Your Model', response.content)

    def test_json_rpc(self):
        self.authenticate('admin', 'admin')
        result = self.make_jsonrpc_request(
            '/web/dataset/call_kw',
            {
                'model': 'your.model',
                'method': 'search_read',
                'args': [[['state', '=', 'confirmed']]],
                'kwargs': {'fields': ['name', 'state'], 'limit': 5},
            }
        )
        self.assertIsInstance(result, list)
```

---

## 7. Tour Tests (JS UI Automation)

### Python side
```python
@tagged('post_install', '-at_install')
class TestYourTour(HttpCase):

    def test_your_module_tour(self):
        self.env['your.model'].create({
            'name': 'Tour Test Record',
            'partner_id': self.env.ref('base.res_partner_1').id,
        })
        self.start_tour(
            '/web#action=your_module.action_your_model',
            'your_module_tour',
            login='admin',
        )
```

### JavaScript tour (static/src/js/tours/your_module_tour.js)
```javascript
/** @odoo-module **/
import { registry } from "@web/core/registry";

registry.category("web_tour.tours").add("your_module_tour", {
    url: "/web#action=your_module.action_your_model",
    steps: () => [
        {
            content: "Open list view",
            trigger: ".o_your_model_list",
        },
        {
            content: "Click New",
            trigger: ".o_list_button_add",
            run: "click",
        },
        {
            content: "Fill name",
            trigger: ".o_field_widget[name='name'] input",
            run: "edit Test Tour Record",
        },
        {
            content: "Save",
            trigger: ".o_form_button_save",
            run: "click",
        },
        {
            content: "Confirm state is Draft",
            trigger: ".o_field_widget[name='state'] .badge:contains('Draft')",
        },
    ],
});
```

```python
# Register tour in manifest assets
'assets': {
    'web.assets_tests': [
        'your_module/static/src/js/tours/your_module_tour.js',
    ],
},
```

---

## 8. Common Assertions Reference

```python
# Value equality
self.assertEqual(a, b)
self.assertNotEqual(a, b)
self.assertTrue(expr)
self.assertFalse(expr)
self.assertIsNone(value)
self.assertIn(item, container)
self.assertAlmostEqual(a, b, places=2)   # floats

# Recordset
self.assertRecordValues(records, [{'field': val}, ...])
self.assertEqual(len(records), 3)
self.assertTrue(records)    # non-empty
self.assertFalse(records)   # empty

# Exceptions
with self.assertRaises(UserError): ...
with self.assertRaises(ValidationError): ...
with self.assertRaises(AccessError): ...

with self.assertRaises(UserError) as cm:
    ...
self.assertIn("Expected text", str(cm.exception))

# Log assertions
with self.assertLogs('your_module', level='WARNING') as cm:
    self.record.action_with_warning()
self.assertIn('Expected warning', cm.output[0])
```

---

## 9. Test Data Best Practices

```python
@classmethod
def setUpClass(cls):
    super().setUpClass()

    # ✅ Disable mail overhead
    cls.env = cls.env(context={
        'mail_create_nolog': True,
        'mail_notrack': True,
        'tracking_disable': True,
    })

    # ✅ Use env.ref() for standard records — no DB write
    cls.partner = cls.env.ref('base.res_partner_1')
    cls.currency_usd = cls.env.ref('base.USD')

    # ✅ Minimal create — only required fields
    cls.product = cls.env['product.product'].create({
        'name': 'Test Product',
        'type': 'service',
    })

    # ✅ Use unique logins to avoid conflicts with existing users
    cls.test_user = cls.env['res.users'].create({
        'name': 'Test User',
        'login': 'test_user_unique@test.local',
        'groups_id': [(4, cls.env.ref('base.group_user').id)],
    })
```

---

## 10. Version Notes

| Feature | v17 | v18 | v19 |
|---------|-----|-----|-----|
| `TransactionCase` | ✅ | ✅ | ✅ |
| `SavepointCase` | ❌ removed | ❌ removed | ❌ removed |
| `HttpCase` | ✅ | ✅ | ✅ |
| `Form` helper | ✅ `odoo.tests.common` | ✅ | ✅ |
| `assertRecordValues` | ✅ | ✅ | ✅ |
| `start_tour` | ✅ `HttpCase` | ✅ | ✅ |
| `make_jsonrpc_request` | ✅ `HttpCase` | ✅ | ✅ |
| Tour step `run: "edit ..."` | ✅ | ✅ | ✅ |
