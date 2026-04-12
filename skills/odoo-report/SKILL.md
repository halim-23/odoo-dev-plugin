---
name: odoo-report
description: >
  Odoo QWeb report development for versions 17.0, 18.0, 19.0 (Community & Enterprise).
  Covers PDF and HTML reports, report actions, ir.actions.report, custom report models,
  paper formats, and report inheritance. Use when user asks about Odoo reports, PDF, QWeb,
  printing, or invokes /odoo-report.
---

Expert Odoo report developer. Target: 17/18/19 CE+EE. Uses wkhtmltopdf for PDF rendering.

## Report Action

```xml
<!-- reports/my_report.xml -->
<odoo>
    <data>
        <report
            id="action_report_my_model"
            name="my.model"
            model="my.model"
            string="My Report"
            report_type="qweb-pdf"
            file="my_module.report_my_model_document"
            print_report_name="'My Report - %s' % object.name"
            attachment="'My_Report_%s' % object.name.replace(' ', '_')"
            attachment_use="False"
            binding_model_id="model_my_model"
        />
    </data>
</odoo>
```

| Attribute | Values |
|-----------|--------|
| `report_type` | `qweb-pdf`, `qweb-html` |
| `attachment` | Python expr → filename to auto-attach |
| `attachment_use` | `True` = return cached attachment if exists |
| `binding_model_id` | binds to Print menu on model |

## Report Template

```xml
<!-- reports/my_report_template.xml -->
<odoo>
    <template id="report_my_model_document">
        <t t-call="web.html_container">
            <t t-foreach="docs" t-as="doc">
                <t t-call="web.external_layout">
                    <div class="page">
                        <div class="row">
                            <div class="col-6">
                                <h2 t-field="doc.name"/>
                                <p>Date: <span t-field="doc.date"/></p>
                            </div>
                            <div class="col-6 text-end">
                                <strong t-field="doc.partner_id.name"/>
                                <div t-field="doc.partner_id" t-options='{"widget": "contact", "fields": ["address"]}'/>
                            </div>
                        </div>

                        <table class="table table-sm mt-4">
                            <thead>
                                <tr>
                                    <th>Product</th>
                                    <th class="text-end">Qty</th>
                                    <th class="text-end">Unit Price</th>
                                    <th class="text-end">Subtotal</th>
                                </tr>
                            </thead>
                            <tbody>
                                <t t-foreach="doc.line_ids" t-as="line">
                                    <tr>
                                        <td><span t-field="line.product_id.name"/></td>
                                        <td class="text-end"><span t-field="line.qty"/></td>
                                        <td class="text-end">
                                            <span t-field="line.price_unit"
                                                  t-options='{"widget": "monetary", "display_currency": doc.currency_id}'/>
                                        </td>
                                        <td class="text-end">
                                            <span t-field="line.subtotal"
                                                  t-options='{"widget": "monetary", "display_currency": doc.currency_id}'/>
                                        </td>
                                    </tr>
                                </t>
                            </tbody>
                            <tfoot>
                                <tr>
                                    <td colspan="3" class="text-end"><strong>Total</strong></td>
                                    <td class="text-end">
                                        <span t-field="doc.amount_total"
                                              t-options='{"widget": "monetary", "display_currency": doc.currency_id}'/>
                                    </td>
                                </tr>
                            </tfoot>
                        </table>

                        <!-- Conditional section (EE feature / custom) -->
                        <div t-if="doc.notes" class="mt-4">
                            <strong>Notes:</strong>
                            <p t-field="doc.notes"/>
                        </div>
                    </div>
                </t>
            </t>
        </t>
    </template>
</odoo>
```

## Custom Report Model (extra data)

```python
# models/report_my_model.py
from odoo import api, models


class ReportMyModel(models.AbstractModel):
    _name = 'report.my_module.report_my_model_document'
    _description = 'Report My Model'

    @api.model
    def _get_report_values(self, docids, data=None):
        docs = self.env['my.model'].browse(docids)
        return {
            'doc_ids': docids,
            'doc_model': 'my.model',
            'docs': docs,
            'data': data,
            'company': self.env.company,
        }
```

Use in template: `t-foreach="docs"` same as above. Access `company`, `data` via template vars.

## Paper Format

```xml
<record id="paperformat_my_module" model="report.paperformat">
    <field name="name">My Custom Format</field>
    <field name="default" eval="False"/>
    <field name="format">A4</field>
    <field name="orientation">Portrait</field>
    <field name="margin_top">25</field>
    <field name="margin_bottom">20</field>
    <field name="margin_left">7</field>
    <field name="margin_right">7</field>
    <field name="header_line" eval="False"/>
    <field name="header_spacing">35</field>
    <field name="dpi">90</field>
</record>

<!-- Bind to action -->
<record id="action_report_my_model" model="ir.actions.report">
    <field name="paperformat_id" ref="paperformat_my_module"/>
</record>
```

## Print from Python

```python
def action_print_report(self):
    return self.env.ref('my_module.action_report_my_model').report_action(self)

# Print multiple with extra data
def action_print_custom(self):
    data = {'custom_key': 'value'}
    return self.env.ref('my_module.action_report_my_model').report_action(self, data=data)
```

## Inherit Existing Report

```xml
<template id="report_invoice_custom" inherit_id="account.report_invoice_document">
    <xpath expr="//div[@class='page']" position="inside">
        <div t-if="doc.my_custom_field" class="mt-2">
            <strong>Custom Field:</strong> <span t-field="doc.my_custom_field"/>
        </div>
    </xpath>
</template>
```

## Version Notes

**v17:** `web.external_layout` includes company logo/address automatically.
**v17:** Bootstrap **5.1.3** used in report templates (was BS4 in v16).
**v18:** Bootstrap **5.3.3** (updated from 5.1.3). `report_type="qweb-pdf-no-barcode"` for faster rendering without barcode libs.
**v18/v19:** wkhtmltopdf remains the PDF engine across all versions (source-verified v17/18/19).

## Common t-options Widgets

```xml
<span t-field="doc.date" t-options='{"widget": "date"}'/>
<span t-field="doc.datetime" t-options='{"widget": "datetime"}'/>
<span t-field="doc.amount" t-options='{"widget": "monetary", "display_currency": doc.currency_id}'/>
<div t-field="doc.partner_id" t-options='{"widget": "contact", "fields": ["name", "address", "phone", "email"]}'/>
```
