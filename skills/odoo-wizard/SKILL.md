---
name: odoo-wizard
description: >
  Odoo wizard (transient model) development for versions 17.0, 18.0, 19.0 (Community & Enterprise).
  Covers TransientModel, wizard views, context passing, multi-record actions, and return actions.
  Use when user asks about Odoo wizards, popups, transient models, or invokes /odoo-wizard.
---

Expert Odoo wizard developer. Target: 17/18/19 CE+EE.

## Wizard Model

```python
from odoo import api, fields, models, _
from odoo.exceptions import UserError


class MyWizard(models.TransientModel):
    _name = 'my.wizard'
    _description = 'My Wizard'

    # Pre-filled from context
    record_ids = fields.Many2many(
        'my.model', string='Records',
        default=lambda self: self.env.context.get('active_ids', [])
    )
    date = fields.Date(string='Date', required=True, default=fields.Date.today)
    reason = fields.Text(string='Reason')
    action_type = fields.Selection([
        ('confirm', 'Confirm'),
        ('cancel', 'Cancel'),
    ], string='Action', required=True, default='confirm')

    def action_execute(self):
        self.ensure_one()
        if not self.record_ids:
            raise UserError(_('No records selected.'))

        for rec in self.record_ids:
            if self.action_type == 'confirm':
                rec.with_context(wizard_date=self.date).action_confirm()
            elif self.action_type == 'cancel':
                rec.action_cancel()

        # Return action to refresh / close
        return {'type': 'ir.actions.act_window_close'}

    def action_execute_and_open(self):
        """Execute and open result in a list view."""
        self.action_execute()
        return {
            'type': 'ir.actions.act_window',
            'name': _('Processed Records'),
            'res_model': 'my.model',
            'domain': [('id', 'in', self.record_ids.ids)],
            'view_mode': 'list,form',
            'target': 'current',
        }
```

## Wizard View

```xml
<record id="view_my_wizard_form" model="ir.ui.view">
    <field name="name">my.wizard.form</field>
    <field name="model">my.wizard</field>
    <field name="arch" type="xml">
        <form string="My Wizard">
            <group>
                <field name="record_ids" widget="many2many_tags"/>
                <field name="date"/>
                <field name="action_type"/>
                <field name="reason" invisible="action_type != 'cancel'"/>
            </group>
            <footer>
                <button name="action_execute" string="Execute" type="object"
                        class="btn-primary"/>
                <button string="Cancel" class="btn-secondary" special="cancel"/>
            </footer>
        </form>
    </field>
</record>
```

## Wizard Action (opens as dialog)

```xml
<record id="action_my_wizard" model="ir.actions.act_window">
    <field name="name">My Wizard</field>
    <field name="res_model">my.wizard</field>
    <field name="view_mode">form</field>
    <field name="target">new</field>          <!-- opens as modal dialog -->
    <field name="binding_model_id" ref="model_my_model"/>  <!-- binds to Action menu -->
    <field name="binding_view_types">list,form</field>
</record>
```

## Call Wizard from Python

```python
def action_open_wizard(self):
    """Open wizard pre-filled with current record(s)."""
    return {
        'type': 'ir.actions.act_window',
        'name': _('Process Records'),
        'res_model': 'my.wizard',
        'view_mode': 'form',
        'target': 'new',
        'context': {
            'default_record_ids': [(6, 0, self.ids)],
            'default_action_type': 'confirm',
        },
    }
```

## Read Context in Wizard

```python
@api.model
def default_get(self, fields_list):
    res = super().default_get(fields_list)
    active_ids = self.env.context.get('active_ids', [])
    active_model = self.env.context.get('active_model')
    if active_model == 'my.model' and active_ids:
        res['record_ids'] = [(6, 0, active_ids)]
    return res
```

## Onchange in Wizard

```python
@api.onchange('action_type')
def _onchange_action_type(self):
    if self.action_type == 'confirm':
        self.reason = False
    return {
        'warning': {
            'title': _('Warning'),
            'message': _('This will confirm all selected records.'),
        }
    }  # return warning dict OR False; do NOT write to DB in onchange
```

## Version Notes

**v17:** `footer` in wizard form auto-renders as sticky bottom bar in dialog.
**v17:** `target='new'` opens full-screen dialog on mobile, modal on desktop.
**v18:** `target='new'` dialogs support `size` attribute: `extra-large`, `large`, `medium` (default), `small`.
**v19:** `TransientModel` records GC interval configurable per model via `_transient_max_hours`.

## Best Practices

- Always `ensure_one()` in execute methods unless explicitly multi
- Use `(6, 0, ids)` for M2M default_get — never raw list
- Return `{'type': 'ir.actions.act_window_close'}` to close dialog
- For batch action from list view: bind via `binding_model_id` — no server action needed
- Clean up transient state: Odoo auto-vacuums TransientModel records after `_transient_max_hours` (default 1h)
