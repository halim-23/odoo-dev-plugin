---
name: odoo-security
description: >
  Odoo security configuration for versions 17.0, 18.0, 19.0 (Community & Enterprise).
  Covers ir.model.access.csv (ACL), record rules, security groups, field-level access,
  and multi-company rules. Use when user asks about Odoo security, access rights, record rules,
  groups, or invokes /odoo-security.
---

Expert Odoo security engineer. Target: 17/18/19 CE+EE.

## ACL — ir.model.access.csv

```csv
id,name,model_id:id,group_id:id,perm_read,perm_write,perm_create,perm_unlink
access_my_model_user,my.model user,model_my_model,my_module.group_my_module_user,1,1,1,0
access_my_model_manager,my.model manager,model_my_model,my_module.group_my_module_manager,1,1,1,1
access_my_model_public,my.model public,model_my_model,,1,0,0,0
```

**Rules:**
- `model_id:id` = `model_` + model name with dots replaced by `_`
- `group_id:id` blank = every user (including portal/public)
- Permissions: 1=yes, 0=no

## Security Groups

```xml
<!-- security/security.xml -->
<odoo>
    <data>
        <!-- Category (shown in Settings > Users) -->
        <record id="module_category_my_module" model="ir.module.category">
            <field name="name">My Module</field>
            <field name="sequence">100</field>
        </record>

        <!-- User group -->
        <record id="group_my_module_user" model="res.groups">
            <field name="name">User</field>
            <field name="category_id" ref="module_category_my_module"/>
            <field name="implied_ids" eval="[(4, ref('base.group_user'))]"/>
        </record>

        <!-- Manager group (implies User) -->
        <record id="group_my_module_manager" model="res.groups">
            <field name="name">Manager</field>
            <field name="category_id" ref="module_category_my_module"/>
            <field name="implied_ids" eval="[(4, ref('group_my_module_user'))]"/>
            <field name="users" eval="[(4, ref('base.user_admin'))]"/>
        </record>
    </data>
</odoo>
```

## Record Rules

```xml
<!-- Multi-company rule — own company only -->
<record id="rule_my_model_company" model="ir.rule">
    <field name="name">My Model: Company</field>
    <field name="model_id" ref="model_my_model"/>
    <field name="domain_force">[('company_id', 'in', company_ids)]</field>
    <field name="groups" eval="[(4, ref('base.group_user'))]"/>
    <field name="perm_read" eval="True"/>
    <field name="perm_write" eval="True"/>
    <field name="perm_create" eval="True"/>
    <field name="perm_unlink" eval="True"/>
</record>

<!-- Personal records — see own records only -->
<record id="rule_my_model_personal" model="ir.rule">
    <field name="name">My Model: Personal</field>
    <field name="model_id" ref="model_my_model"/>
    <field name="domain_force">[('user_id', '=', user.id)]</field>
    <field name="groups" eval="[(4, ref('group_my_module_user'))]"/>
</record>

<!-- Manager sees all — global rule (no group = applies to all) -->
<record id="rule_my_model_all" model="ir.rule">
    <field name="name">My Model: All</field>
    <field name="model_id" ref="model_my_model"/>
    <field name="domain_force">[(1, '=', 1)]</field>
    <field name="groups" eval="[(4, ref('group_my_module_manager'))]"/>
</record>
```

**Rule evaluation:** multiple rules for same group → OR'd together. Rules across groups → AND'd.

## Field-Level Security

```python
# On model field
class MyModel(models.Model):
    _name = 'my.model'

    # Visible only to managers
    cost_price = fields.Float(
        groups='my_module.group_my_module_manager'
    )

    # Read-only for users, editable for managers
    state = fields.Selection(
        [...],
        groups='my_module.group_my_module_manager'  # hides from non-managers
    )
```

In XML view:
```xml
<field name="cost_price" groups="my_module.group_my_module_manager"/>
```

## Group Usage in Views / Menus

```xml
<!-- Hide menu for non-managers -->
<menuitem id="menu_config" name="Configuration"
          parent="menu_root"
          groups="my_module.group_my_module_manager"
          sequence="99"/>

<!-- Hide button for users -->
<button name="action_delete_all" string="Delete All" type="object"
        groups="my_module.group_my_module_manager"/>
```

## Check Access in Python

```python
def action_dangerous(self):
    self.ensure_one()
    # Check group
    if not self.env.user.has_group('my_module.group_my_module_manager'):
        raise AccessError(_('Only managers can perform this action.'))
    # Check model access
    self.env['my.model'].check_access_rights('unlink')
    self.env['my.model'].browse(self.id).check_access_rule('unlink')
```

## Portal / Public Access

```python
# Grant portal users read access in CSV:
# access_my_model_portal,my.model portal,model_my_model,base.group_portal,1,0,0,0

# In controller, filter by partner
@http.route('/portal/my_model', auth='user', website=True)
def portal_list(self):
    domain = [('partner_id', '=', request.env.user.partner_id.id)]
    records = request.env['my.model'].search(domain)
    return request.render('my_module.portal_template', {'records': records})
```

## Version Notes

**v17:** `column_invisible` attribute on list view columns (replaces complex attrs).
**v17:** `invisible` on `<field>` accepts direct Python expression — no `groups` needed for UI hide.
**v18:** `ir.rule` supports `active` field — deactivate without deleting.
**v19:** `res.groups` gains `color` field for UI badge display.

## Multi-Company Setup

```python
# Add to model
class MyModel(models.Model):
    _name = 'my.model'

    company_id = fields.Many2one(
        'res.company', string='Company',
        required=True,
        default=lambda self: self.env.company
    )

# Record rule ensures users see only their company's records
```

## Security Checklist

- [ ] `ir.model.access.csv` for every new model
- [ ] Record rules for multi-company models
- [ ] `sudo()` only when intentional — comment why
- [ ] Never trust user input in domain/eval without sanitization
- [ ] Field `groups=` for sensitive financial fields
- [ ] Portal ACL if portal users need access
- [ ] `check_access_rights` + `check_access_rule` before dangerous ops
