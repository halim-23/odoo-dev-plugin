---
name: odoo-accounting
description: >
  Expert Odoo accounting module customization for versions 17.0, 18.0, 19.0 (Community & Enterprise).
  Covers account.move (invoices, bills, journal entries), account.move.line, account.payment,
  account.tax, account.fiscal.position, analytic distribution, reconciliation, chart of accounts,
  and payment terms. Use when user mentions invoices, bills, payments, journal entries,
  reconciliation, taxes, fiscal positions, analytic accounts, or invokes /odoo-accounting.
---

Expert Odoo accounting engineer. Target: 17/18/19 CE+EE.

**Always confirm Odoo version** — field names differ across versions (see Version Matrix below).

---

## Core Model Map

```
account.move              → Invoice / Bill / Journal Entry (all one model)
account.move.line         → Journal items / invoice product lines
account.payment           → Customer & vendor payments
account.journal           → Sales, Purchase, Bank, Cash, Misc
account.account           → Chart of accounts (individual GL accounts)
account.tax               → Tax definitions
account.fiscal.position   → Tax/account mapping by country or partner
account.payment.term      → Payment terms (net 30, 2/10 net 30, etc.)
account.analytic.account  → Cost centres / analytic axes
```

---

## 1. account.move — Invoice / Bill / Journal Entry

### move_type values (source-verified v17/v18/v19)
```python
'out_invoice'    # Customer Invoice
'out_refund'     # Customer Credit Note
'in_invoice'     # Vendor Bill
'in_refund'      # Vendor Credit Note
'entry'          # Manual Journal Entry
```

### Create a customer invoice
```python
from odoo import models, fields, api, _
from odoo.exceptions import UserError


class YourModel(models.Model):
    _name = 'your.model'

    def action_create_invoice(self):
        self.ensure_one()

        invoice_line_vals = []
        for line in self.line_ids:
            invoice_line_vals.append((0, 0, {
                'product_id': line.product_id.id,
                'name': line.description or line.product_id.name,
                'quantity': line.quantity,
                'price_unit': line.price_unit,
                'account_id': (
                    line.product_id.property_account_income_id.id
                    or self._get_default_income_account().id
                ),
                # tax_ids confirmed Many2many in v17/v18/v19 on account.move.line
                'tax_ids': [(6, 0, line.product_id.taxes_id.ids)],
                # analytic_distribution: JSON field {str(analytic_account_id): percentage}
                # Available v17+ (replaces old analytic_account_id on lines)
                'analytic_distribution': (
                    {str(self.analytic_account_id.id): 100}
                    if self.analytic_account_id else {}
                ),
            }))

        invoice = self.env['account.move'].create({
            'move_type': 'out_invoice',
            'partner_id': self.partner_id.id,
            'invoice_date': fields.Date.today(),
            'invoice_date_due': self._compute_due_date(),
            'invoice_origin': self.name,
            'ref': self.reference,
            'journal_id': self._get_sales_journal().id,
            'currency_id': self.currency_id.id,
            'invoice_line_ids': invoice_line_vals,
            'narration': self.notes,
        })
        self.invoice_id = invoice.id
        return {
            'type': 'ir.actions.act_window',
            'res_model': 'account.move',
            'res_id': invoice.id,
            'view_mode': 'form',
            'target': 'current',
        }

    def _get_default_income_account(self):
        return self.env['account.account'].search([
            ('account_type', '=', 'income'),
            ('company_id', '=', self.env.company.id),
        ], limit=1)

    def _get_sales_journal(self):
        return self.env['account.journal'].search([
            ('type', '=', 'sale'),
            ('company_id', '=', self.env.company.id),
        ], limit=1)

    def _compute_due_date(self):
        term = self.partner_id.property_payment_term_id
        if term:
            lines = term._compute_terms(
                date_ref=fields.Date.today(),
                currency=self.currency_id,
                company=self.env.company,
                tax_amount=0,
                tax_amount_currency=0,
                sign=1,
                untaxed_amount=self.amount_total,
                untaxed_amount_currency=self.amount_total,
            )
            return lines[-1]['date'] if lines else fields.Date.today()
        return fields.Date.today()
```

### Invoice lifecycle actions (source-verified v18)
```python
# Draft → Posted
invoice.action_post()

# Posted → Draft (reset)
invoice.button_draft()

# Posted → Cancelled
invoice.button_cancel()

# Check state
if invoice.state == 'draft':
    invoice.action_post()
```

### Read invoice data
```python
invoice.name              # e.g. INV/2024/00001
invoice.state             # 'draft', 'posted', 'cancel'
invoice.payment_state     # see Payment State values below
invoice.move_type         # 'out_invoice', etc.
invoice.partner_id
invoice.invoice_date
invoice.invoice_date_due
invoice.amount_untaxed
invoice.amount_tax
invoice.amount_total
invoice.amount_residual   # unpaid balance
invoice.currency_id
invoice.invoice_line_ids  # product lines (display_type=False)
invoice.line_ids          # all journal items (incl. tax/payable lines)
```

### Payment state values (source-verified v18 addons/account/models/account_move.py:51)
```python
'not_paid'          # Not paid
'in_payment'        # In Payment (payment posted, not yet bank-matched)
'paid'              # Fully paid
'partial'           # Partially paid
'reversed'          # Reversed by a credit note
'blocked'           # Blocked (do not follow up)
'invoicing_legacy'  # Legacy invoicing app state
```

---

## 2. account.move.line — Journal Items

```python
# Invoice product lines only
product_lines = invoice.invoice_line_ids

# All journal items (tax lines, payable/receivable lines too)
all_lines = invoice.line_ids

# Filter by account type
receivable_lines = invoice.line_ids.filtered(
    lambda l: l.account_id.account_type == 'asset_receivable'
)
tax_lines = invoice.line_ids.filtered(lambda l: l.tax_line_id)

# Create a manual journal entry (balanced debit/credit required)
entry = self.env['account.move'].create({
    'move_type': 'entry',
    'journal_id': misc_journal.id,
    'date': fields.Date.today(),
    'ref': 'Manual adjustment',
    'line_ids': [
        (0, 0, {
            'account_id': debit_account.id,
            'name': 'Debit line',
            'debit': 1000.0,
            'credit': 0.0,
            'partner_id': partner.id,
        }),
        (0, 0, {
            'account_id': credit_account.id,
            'name': 'Credit line',
            'debit': 0.0,
            'credit': 1000.0,
            'partner_id': partner.id,
        }),
    ],
})
entry.action_post()
```

---

## 3. account.payment — Register Payments

```python
# Register payment directly (without wizard)
payment = self.env['account.payment'].create({
    'payment_type': 'inbound',      # 'inbound'=receive money, 'outbound'=send
    'partner_type': 'customer',     # 'customer' or 'supplier'
    'partner_id': invoice.partner_id.id,
    'amount': invoice.amount_residual,
    'currency_id': invoice.currency_id.id,
    'journal_id': bank_journal.id,
    'date': fields.Date.today(),
    'ref': invoice.name,
})
payment.action_post()

# Reconcile payment with invoice
(invoice + payment.move_id).line_ids.filtered(
    lambda l: l.account_id.account_type in (
        'asset_receivable', 'liability_payable'
    ) and not l.reconciled
).reconcile()
```

---

## 4. Extending account.move

```python
# models/account_move.py
class AccountMove(models.Model):
    _inherit = 'account.move'

    your_custom_field = fields.Char(string='Custom Reference')
    project_id = fields.Many2one('project.project', string='Project')

    @api.onchange('partner_id')
    def _onchange_partner_custom(self):
        if self.partner_id and self.partner_id.default_project_id:
            self.project_id = self.partner_id.default_project_id
```

```xml
<!-- views/account_move_views.xml -->
<record id="view_move_form_inherit_your_module" model="ir.ui.view">
    <field name="name">account.move.form.inherit.your_module</field>
    <field name="model">account.move</field>
    <field name="inherit_id" ref="account.view_move_form"/>
    <field name="arch" type="xml">
        <field name="partner_id" position="after">
            <field name="your_custom_field"
                   invisible="move_type not in ('out_invoice','out_refund')"/>
            <field name="project_id"/>
        </field>
    </field>
</record>
```

---

## 5. Taxes

### Compute tax on a product
```python
product = self.env['product.product'].browse(product_id)
partner = self.env['res.partner'].browse(partner_id)

# Customer taxes
taxes = product.taxes_id   # taxes_id is Many2many on product.template (all versions)

# Apply fiscal position mapping
fiscal_pos = partner.property_account_position_id
if fiscal_pos:
    taxes = fiscal_pos.map_tax(taxes)

# Compute tax amounts
tax_values = taxes.compute_all(
    price_unit=100.0,
    currency=self.env.company.currency_id,
    quantity=2,
    product=product,
    partner=partner,
)
# Returns: {'total_excluded': 200.0, 'total_included': 230.0, 'taxes': [...]}
```

### Create a tax in XML data
```xml
<record id="tax_your_custom_10" model="account.tax">
    <field name="name">Custom Tax 10%</field>
    <field name="type_tax_use">sale</field>   <!-- sale, purchase, none -->
    <field name="amount_type">percent</field> <!-- percent, fixed, division -->
    <field name="amount">10</field>
    <field name="price_include" eval="False"/>
    <field name="company_id" ref="base.main_company"/>
</record>
```

---

## 6. Fiscal Positions

```python
# Get fiscal position for a partner
fiscal_pos = partner.property_account_position_id
# Or auto-detect:
fiscal_pos = self.env['account.fiscal.position'].get_fiscal_position(partner_id)

# Map taxes and accounts
mapped_taxes = fiscal_pos.map_tax(product.taxes_id) if fiscal_pos else product.taxes_id
mapped_account = fiscal_pos.map_account(account) if fiscal_pos else account
```

---

## 7. Analytic Distribution (v17/v18/v19)

`analytic_distribution` replaces `analytic_account_id` on journal lines in v16+.

```python
# Set analytic on a move line — JSON dict {str(account_id): percentage}
move_line_vals = {
    'account_id': account.id,
    'name': 'Service',
    'debit': 500.0,
    'analytic_distribution': {str(analytic_account.id): 100},
    # Split: {str(account_1.id): 60, str(account_2.id): 40}
}

# Read analytic distribution
for account_id_str, percentage in (line.analytic_distribution or {}).items():
    account = self.env['account.analytic.account'].browse(int(account_id_str))
    print(account.name, percentage)
```

---

## 8. Common Search Patterns

```python
# Unpaid customer invoices for a partner
unpaid = self.env['account.move'].search([
    ('partner_id', '=', partner_id),
    ('move_type', '=', 'out_invoice'),
    ('state', '=', 'posted'),
    ('payment_state', 'in', ['not_paid', 'partial']),
])

# Account by code
account = self.env['account.account'].search([
    ('code', '=', '400000'),
    ('company_id', '=', self.env.company.id),
], limit=1)

# Journal by type
sales_journal = self.env['account.journal'].search([
    ('type', '=', 'sale'),
    ('company_id', '=', self.env.company.id),
], limit=1)

# Check if accounting module is installed
has_accounting = bool(
    self.env['ir.module.module'].search([
        ('name', '=', 'account'), ('state', '=', 'installed'),
    ])
)
```

---

## 9. Version Differences Matrix

| Feature | v17 | v18 | v19 |
|---------|-----|-----|-----|
| `tax_ids` on `account.move.line` | ✅ Many2many | ✅ Many2many | ✅ Many2many |
| `taxes_id` on `product.template` | ✅ | ✅ | ✅ |
| `analytic_distribution` (JSON) | ✅ | ✅ | ✅ |
| `tax_id` on `sale.order.line` | ✅ | ✅ | renamed `tax_ids` |
| `account_type` string enum | ✅ | ✅ | ✅ |
| `payment_state` values | same | same | same |
| `action_register_payment()` | ✅ | ✅ | ✅ |

---

## 10. Common Errors & Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `You cannot post a journal entry without an account` | `account_id` missing on a line | Set `account_id` on every move line |
| `The journal entry is not balanced` | Debit ≠ Credit totals | Verify all line amounts sum to same |
| `You cannot delete a posted journal entry` | Must reset to draft first | `button_draft()` then `unlink()` |
| `Fiscal year not found` | No fiscal year configured | Settings → Accounting → Fiscal Years |
| `Tax account not found` | Tax repartition missing account | Configure tax repartition lines |
