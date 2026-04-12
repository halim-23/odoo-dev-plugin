---
name: odoo-mail-template
description: >
  Odoo mail.template (ir.mail_template) development for v17.0, v18.0, v19.0 (CE + EE).
  Source-verified knowledge: field changes, method API, render engine (QWeb + inline template),
  email layout system, dynamic placeholders, security model, send_mail/generate_email,
  scheduled messages, canned responses, HTML editor changes, preview wizard.
  Use when user asks about mail templates, email templates, email layouts, dynamic placeholders,
  send_mail, generate_email, or invokes /odoo-mail-template.
---

Expert Odoo mail template developer. All knowledge source-verified against /opt/developments/ v17/18/19.

---

## `mail.template` Model Fields — Version Matrix

### Core fields (all versions)

| Field | Type | Notes |
|-------|------|-------|
| `name` | Char | translatable |
| `description` | Text | translatable |
| `active` | Boolean | default=True; archive to hide from list |
| `model_id` | Many2one(`ir.model`) | required; v19: domain excludes abstract models |
| `model` | Char | related to model_id.model; use for domain in actions |
| `subject` | Char | translatable, prefetch=True |
| `email_from` | Char | v19 label: "Send From" (was "From") |
| `user_id` | Many2one(`res.users`) | v19 label: "Owner" (was "User") |
| `use_default_to` | Boolean | v19: `default=True`; v19 label: "Default Recipients" |
| `email_to` | Char | comma-separated |
| `partner_to` | Char | comma-separated record ids or XML IDs |
| `email_cc` | Char | |
| `reply_to` | Char | |
| `body_html` | Html | render_engine changes per version (see below) |
| `attachment_ids` | Many2many(`ir.attachment`) | v19: `bypass_search_access=True` |
| `report_template_ids` | Many2many(`ir.actions.report`) | v19: only shown if `has_dynamic_reports` |
| `email_layout_xmlid` | Char | XML ID of layout template (e.g. `mail.mail_notification_layout`) |
| `mail_server_id` | Many2one(`ir.mail_server`) | v19: `index='btree_not_null'`; v19: only shown if `has_mail_server` |
| `scheduled_date` | Char | Jinja2 expression; v19 placeholder: "Send Instantly" |
| `auto_delete` | Boolean | default=True |
| `lang` | Char | Jinja2 expression; v19 help: "main partner's language will be used" |

### Fields added per version

| Field | Added | Description |
|-------|-------|-------------|
| `template_category` | v17 | Selection: `base_template`, `hidden_template`, `custom_template`; computed |
| `can_write` | v17 | Boolean computed; controls edit access |
| `is_template_editor` | v17 | Boolean computed; True if user in mail.group_mail_template_editor |
| `has_dynamic_reports` | v19 | Boolean computed; True if model has dynamic reports |
| `has_mail_server` | v19 | Boolean computed; True if any mail server configured |

---

## `body_html` Field — Critical Changes

```python
# v17 — NO sanitization (raw HTML allowed)
body_html = fields.Html(
    render_engine='qweb',
    render_options={'post_process': True},
    sanitize=False,
)

# v18 — sanitize changed to 'email_outgoing'
body_html = fields.Html(
    render_engine='qweb',
    render_options={'post_process': True},
    sanitize='email_outgoing',   # ← CHANGED: strips scripts, on* attrs, etc.
)

# v19 — same as v18
body_html = fields.Html(
    render_engine='qweb',
    render_options={'post_process': True},
    sanitize='email_outgoing',
)
```

**Impact**: v17 templates with raw `<script>` tags or `on*` attributes will fail validation in v18+.

---

## HTML Editor Widget — View Changes

```xml
<!-- v17: standard html widget -->
<field name="body_html" widget="html"/>

<!-- v18: html_mail widget + /field command support + video disabled -->
<field name="body_html" widget="html_mail"
       options="{'disableVideo': true}"/>
<!-- Helper hint shown below editor: "Write /field to insert dynamic content" -->

<!-- v19: html_mail widget + updated option name -->
<field name="body_html" widget="html_mail"
       options="{'allowCommandVideo': false}"/>
<!-- Tip: "Write /field to insert dynamic content!" -->
```

---

## Form View Structure — Version Changes

### v17 pages
1. **Content** — body_html editor
2. **Email Configuration** — email_from, email_to, email_cc, reply_to, partner_to, lang, scheduled_date
3. **Settings** — mail_server, auto_delete, reports, user_id, description

### v18 pages
1. **Content** — body_html (`html_mail` widget), /field hint
2. **Email Configuration** — same fields
3. **Settings** — same; `groups="base.group_system"` on admin buttons (was `group_no_one`)

### v19 pages (major restructure)
1. **Body** (was Content) — body_html, "Write your message here…" placeholder
2. **Settings** (`groups="base.group_no_one"`) — two separators:
   - *Sender & Recipients*: email_from, use_default_to, email_to, partner_to, email_cc, reply_to, lang
   - *Technical*: mail_server_id (invisible if `not has_mail_server`), scheduled_date, auto_delete
3. **Options** — attachment_ids, report_template_ids (invisible if `not has_dynamic_reports`), description

---

## Methods API

### `send_mail()` — send one email

```python
# All versions — signature
template.send_mail(
    res_id,
    force_send=False,
    raise_exception=False,
    email_layout_xmlid=None,   # e.g. 'mail.mail_notification_light'
    email_values=None,         # dict to override generated values
    notif_layout=None,         # deprecated alias for email_layout_xmlid
)
# Returns mail.mail id
```

### `send_mail_batch()` — v17+ batch send

```python
# v17+ — batch version with @api.returns decorator
template.send_mail_batch(
    res_ids,                   # list of record IDs
    force_send=False,
    raise_exception=False,
    email_layout_xmlid=None,
    email_values=None,
)
# Returns recordset of mail.mail
```

### `generate_email()` — generate values dict without sending

```python
# All versions
values = template.generate_email(
    res_ids,                   # int or list
    fields=None,               # list of fields to render, default: all
)
# Returns dict {res_id: {subject, body_html, email_to, ...}}
```

### `_generate_template()` — internal orchestrator

```python
# Called by generate_email(); broken into sub-methods:
# - _generate_template_recipients()    — email_to, partner_to, email_cc
# - _generate_template_attachments()   — attachment_ids + report_template_ids
# - _generate_template_scheduled_date() — scheduled_date rendering
# - _generate_template_static_values() — subject, email_from, reply_to, lang, body_html
```

---

## Render Engine — Changes Per Version

### Two render engines in all versions

```python
# 1. QWeb (default for body_html)
body_html = fields.Html(render_engine='qweb')

# 2. Inline template (for text fields: subject, email_from, email_to, etc.)
subject = fields.Char()  # uses inline template engine by default
```

### Inline template syntax (all versions)

```
{{object.name}}                         # field value
{{object.partner_id.name or ''}}        # with fallback
{{format_date(object.date)}}            # format helpers
{{format_amount(object.amount, object.currency_id)}}
{{user.name}}                           # current user
{{ctx.get('default_some_field')}}       # context access
```

### `_build_expression()` — null value operator changed

```python
# v17: uses Python 'or' with triple-quoted fallback
expr = "record.name or '''N/A'''"

# v18+: uses ||| operator (custom separator)
expr = "record.name ||| N/A"
```

### Unsafe expression detection — renamed v18

```python
# v17
template._is_dynamic()
template._is_dynamic_template_qweb()
template._is_dynamic_template_inline_template()

# v18+: RENAMED
template._has_unsafe_expression(model)
template._has_unsafe_expression_template_qweb(source, model)
template._has_unsafe_expression_template_inline_template(source, model)

# v19: fname parameter added
template._has_unsafe_expression_template_qweb(source, model, fname=None)
template._has_unsafe_expression_template_inline_template(source, model, fname=None)
```

### QWebException → QWebError (v19)

```python
# v17/v18
from odoo.addons.base.models.ir_qweb import QWebException

# v19 — RENAMED
from odoo.addons.base.models.ir_qweb import QWebError
```

---

## Dynamic Placeholders — Security Model

### Group: `mail.group_mail_template_editor`

Only members of this group can save templates containing "unsafe" expressions (arbitrary Python code).

### v17 check
```python
# _is_dynamic() returns True → requires template_editor group
# Context used: raise_on_code=True
```

### v18 check (tightened)
```python
# _has_unsafe_expression(model) — uses raise_on_forbidden_code_for_model context
# Expression validation via _is_expression_allowed(expr, model)
# Requires: model parameter passed to all checks
```

### v19 check (exemptions added)
```python
# _expression_is_default(source, fname) — NEW
# Returns True if source matches model._mail_template_default_values()[fname]
# Default values are EXEMPT from template_editor group requirement

# create() and write() now call _check_can_be_rendered() to validate
# rendering BEFORE saving — catches broken templates early
```

### `_mail_template_default_values()` — v19 model hook

```python
class MyModel(models.Model):
    _name = 'my.model'

    def _mail_template_default_values(self):
        """Return default values for mail template fields.
        Called when a template's model_id is changed.
        Also used to exempt expressions from security checks.
        """
        return {
            'subject': "{{ object.name }}",
            'body_html': "<p>Dear {{ object.partner_id.name }},</p>",
            'email_to': "{{ object.partner_id.email }}",
        }
```

---

## Email Layout Templates

### Available layouts (all versions)

```xml
<!-- Full notification layout (default) -->
email_layout_xmlid = 'mail.mail_notification_layout'

<!-- Light layout (no footer, minimal) -->
email_layout_xmlid = 'mail.mail_notification_light'

<!-- With responsible signature (adds user signature) -->
email_layout_xmlid = 'mail.mail_notification_layout_with_responsible_signature'
```

### Layout changes — critical v18 removals

| Feature | v17 | v18 | v19 |
|---------|-----|-----|-----|
| Company logo in header | ✅ shown | ❌ removed | ❌ removed |
| Action links block | ✅ shown | ❌ removed | ❌ removed |
| Internal comms banner | ✅ shown | ❌ removed | ❌ removed |
| Button font-weight | 400 (normal) | **bold** | **bold** |
| Header conditional | always shown | `show_header` variable | `show_header` variable |
| Footer conditional | always shown | `show_footer` variable | `show_footer` variable |
| Unfollow link | always shown | always shown | `t-if="show_unfollow"` |
| Button color | hard-coded | hard-coded | `company.email_primary_color` |
| Tracking "None" label | empty | shown as "None" | shown as "None" |

### Layout context variables (v18+)

```python
# Control header/footer visibility via context when sending
template.send_mail(
    res_id,
    email_values={
        'email_layout_xmlid': 'mail.mail_notification_layout',
    }
)

# In layout QWeb, these variables control rendering:
# show_header — True if email_notification_force_header OR
#               (email_notification_allow_header AND has_button_access)
# show_footer — True if show_header AND author_user._is_internal()
# show_unfollow — v19: explicit control; v17/v18: always shown in footer
```

### Custom email layout

```xml
<!-- my_module/data/mail_layouts.xml -->
<odoo>
    <template id="my_custom_layout" inherit_id="mail.mail_notification_layout">
        <xpath expr="//div[@t-if='show_header']" position="before">
            <div style="background: #f00; padding: 10px;">
                Custom Header Banner
            </div>
        </xpath>
    </template>
</odoo>

<!-- Reference in template or send_mail call -->
<field name="email_layout_xmlid">my_module.my_custom_layout</field>
```

---

## Scheduled Messages — New in v18

v18 introduced `mail.scheduled.message` model (in `mail/models/mail_scheduled_message.py`).

```python
# Schedule a message to be sent at a specific date
self.env['mail.scheduled.message'].create({
    'model': 'sale.order',
    'res_id': order.id,
    'subject': 'Your order is ready',
    'body': '<p>Hello!</p>',
    'scheduled_date': '2025-01-15 10:00:00',
    'mail_template_id': template.id,   # optional: link to template
})
```

---

## Canned Responses — New in v18

v18 introduced `mail.canned.response` model.

```python
# Canned responses for quick reply in chatter
self.env['mail.canned.response'].create({
    'source': '/greeting',         # trigger shortcut
    'substitution': 'Hello, thank you for reaching out. How can I help you?',
})
```

---

## `mail.template` in XML Data Files

### REQUIRED: always wrap in `<data noupdate="1">`

```xml
<odoo>
    <data noupdate="1">   <!-- noupdate=1 preserves user edits on upgrade -->
        <record id="..." model="mail.template">
            ...
        </record>
    </data>
</odoo>
```

Without `noupdate="1"`, every `odoo -u` overwrites user-edited templates.

---

### `t-out` vs `t-esc` — use `t-out` always in mail templates

```xml
<!-- ❌ t-esc — escapes HTML, renders as plain text (wrong for HTML fields) -->
<t t-esc="object.name"/>

<!-- ✅ t-out — renders HTML content; use this always -->
<t t-out="object.name or ''">Fallback preview text</t>
```

Content between `<t t-out>` tags = **preview/sample text** shown in the template editor UI.
Always include it — helps users understand what the field renders.
`t-esc` = 0 usages in Odoo core mail templates (v17/18/19).

---

### v17 / v18 XML pattern (source-verified: sale/data/mail_template_data.xml)

```xml
<odoo>
    <data noupdate="1">
        <record id="email_template_my_model" model="mail.template">
            <field name="name">My Model: Notification</field>
            <field name="model_id" ref="my_module.model_my_model"/>
            <field name="subject">{{ object.company_id.name }} - {{ object.name or 'n/a' }}</field>
            <field name="email_from">{{ (object.user_id.email_formatted or user.email_formatted) }}</field>
            <field name="partner_to">{{ object.partner_id.id }}</field>
            <field name="lang">{{ object.partner_id.lang }}</field>
            <field name="email_layout_xmlid">mail.mail_notification_light</field>
            <field name="auto_delete" eval="True"/>
            <field name="body_html" type="html">
                <p>Dear <t t-out="object.partner_id.name or ''">Customer Name</t>,</p>
                <p>
                    Your record
                    <strong><t t-out="object.name or ''">REF001</t></strong>
                    is now confirmed.
                </p>
                <p>Thank you!</p>
            </field>
            <field name="report_template_ids" eval="[(4, ref('my_module.action_report_my_model'))]"/>
        </record>
    </data>
</odoo>
```

Key points for v17/v18:
- `lang` field **required** for partner language: `{{ object.partner_id.lang }}`
- `partner_to` set to partner ID expression
- `body_html` uses `type="html"` attribute — required so Odoo parses as HTML not escaped string
- `report_template_ids` link: `eval="[(4, ref(...))]"` = add without replacing

---

### v19 XML pattern — `lang` removed, `use_default_to` replaces `partner_to`

```xml
<odoo>
    <data noupdate="1">
        <record id="email_template_my_model" model="mail.template">
            <field name="name">My Model: Notification</field>
            <field name="model_id" ref="my_module.model_my_model"/>
            <field name="subject">{{ object.company_id.name }} - {{ object.name or 'n/a' }}</field>
            <field name="email_from">{{ (object.user_id.email_formatted or user.email_formatted) }}</field>
            <!-- v19: partner_to cleared; use_default_to=True resolves recipients via _mail_get_partner_fields() -->
            <field name="partner_to" eval="False"/>
            <field name="use_default_to" eval="True"/>
            <!-- v19: DO NOT set lang field — default automatically uses main partner's language -->
            <field name="email_layout_xmlid">mail.mail_notification_light</field>
            <field name="auto_delete" eval="True"/>
            <field name="body_html" type="html">
                <p>Dear <t t-out="object.partner_id.name or ''">Customer Name</t>,</p>
                <p>
                    Your record
                    <strong><t t-out="object.name or ''">REF001</t></strong>
                    is now confirmed.
                </p>
                <div>--<br/><t t-out="object.user_id.signature or ''">Mitchell Admin</t></div>
            </field>
            <field name="report_template_ids" eval="[(4, ref('my_module.action_report_my_model'))]"/>
        </record>
    </data>
</odoo>
```

Key v19 changes vs v17/v18:
- `lang` field: **OMIT** — v19 auto-uses partner's language; setting it is redundant
- `partner_to eval="False"` + `use_default_to eval="True"` — standard v19 recipient pattern
- Signature: wrapped in `<div>--<br/><t t-out="..."/></div>` (v17/v18: bare `<t t-out>`)
- Source: sale v19 templates have 0 `lang` fields; 4 `use_default_to` fields

---

### Field changes reference for XML declarations

| XML field | v17 | v18 | v19 |
|-----------|-----|-----|-----|
| `<data noupdate="1">` wrapper | required | required | required |
| `body_html type="html"` | required | required | required |
| `lang` | set: `{{ object.partner_id.lang }}` | set: same | **omit** — auto from partner |
| `partner_to` | set: `{{ object.partner_id.id }}` | set: same | `eval="False"` |
| `use_default_to` | not used | not used | `eval="True"` |
| `t-out` vs `t-esc` in body | `t-out` only | `t-out` only | `t-out` only |
| Fallback text in `t-out` | include | include | include |
| Signature div wrapper | bare `<t t-out>` | bare `<t t-out>` | `<div>--<br/><t t-out></div>` |
| `report_template_ids` eval | `[(4, ref(...))]` | `[(4, ref(...))]` | `[(4, ref(...))]` |
| `ctx.get('proforma')` in subject | allowed | allowed | separated into own template |

---

### Sending from Python code

### Sending from Python code

```python
# Preferred pattern (v17+)
template = self.env.ref('my_module.email_template_my_model')
template.send_mail(self.id, force_send=True)

# With layout override
template.send_mail(
    self.id,
    force_send=True,
    email_layout_xmlid='mail.mail_notification_layout',
)

# With value overrides
template.send_mail(
    self.id,
    email_values={
        'email_to': 'custom@example.com',
        'reply_to': False,
    },
)

# Generate values without sending (e.g., for preview or customization)
values = template.generate_email(self.id, fields=['subject', 'body_html', 'email_to'])
# values = {self.id: {'subject': '...', 'body_html': '...', 'email_to': '...'}}
```

---

## Template Preview Wizard

### v17/v18 preview
```python
# Opens wizard via ir.actions.act_window XML reference
# Fields: subject, email_from, email_to, body_html, attachment_ids, partner_ids
# resource_ref: Reference field to select sample record
```

### v19 preview (extended)
```python
# Opens via template.action_open_mail_preview() method (no XML action ref)
# New fields:
#   has_attachments — computed; controls attachment display
#   has_several_languages_installed — computed; controls lang selector
# Better handling of missing/non-existent model in resource_ref compute
```

---

## Template `template_category` Field (v17+)

| Value | Meaning | Who can delete |
|-------|---------|----------------|
| `base_template` | Odoo-provided template (`noupdate=True` XML) | Nobody (protected) |
| `hidden_template` | Technical template (no UI) | System only |
| `custom_template` | User-created or user-modified | Owner or template editor |

```python
# Check in code
if template.template_category == 'base_template':
    raise UserError("Cannot modify base templates directly. Duplicate first.")
```

---

## `copy_data()` — Added v18

```python
# v18+ — safe template duplication (auto-generates unique name)
new_template = template.copy()
# Name becomes: "Original Name (copy)" → "Original Name (copy 2)" etc.
# Implemented via copy_data() override in mail.template
```

---

## Model `_rec_name` for Templates

```python
# mail.template uses 'name' as _rec_name in all versions
# Searching templates by name:
templates = self.env['mail.template'].search([
    ('model', '=', 'sale.order'),
    ('active', '=', True),
])

# v17+: template_category filter in search view
# Filter by: base_template | custom_template | hidden_template
```

---

## `mail.render.mixin` — Inheritance Pattern

All models that render QWeb/inline templates inherit from this mixin.

```python
class MyMailableMixin(models.AbstractModel):
    _name = 'my.mailable.mixin'
    _inherit = ['mail.render.mixin']

    def _render_field(self, field, res_ids, compute_lang=False, **kwargs):
        """Render a template field for given record IDs."""
        return super()._render_field(field, res_ids, compute_lang=compute_lang, **kwargs)
```

### v18 render changes to be aware of
- `_render_template_postprocess()` now requires `model` parameter
- `_render_template_qweb_regex()` — new post-processing hook
- `_render_template_inline_template_regex()` — new post-processing hook
- `prepend_html_content` / `html_normalize` now imported from `odoo.tools.mail`

---

## Deprecation / Breaking Changes Summary

| API | v17 | v18 | v19 | Replacement |
|-----|-----|-----|-----|-------------|
| `body_html sanitize=False` | default | ❌ removed | — | `sanitize='email_outgoing'` |
| `_is_dynamic()` | ✅ | ❌ renamed | — | `_has_unsafe_expression(model)` |
| `_is_dynamic_template_qweb()` | ✅ | ❌ renamed | — | `_has_unsafe_expression_template_qweb(src, model)` |
| `_filter_access_rules('write')` | ✅ | ❌ replaced | — | `_filtered_access('write')` |
| `check_access_rights/rule` | ✅ | ❌ replaced | — | `check_access()` |
| `user_has_groups()` | ✅ | ❌ replaced | — | `env.user.has_group()` |
| QWebException | ✅ | ✅ | ❌ renamed | `QWebError` |
| Company logo in layout | ✅ | ❌ removed | — | Not available |
| Actions block in layout | ✅ | ❌ removed | — | Not available |
| `widget="html"` on body_html | ✅ | ❌ changed | — | `widget="html_mail"` |
| `disableVideo` option | — | ✅ v18 | ❌ renamed | `allowCommandVideo: false` |
| Abstract model as template.model | ✅ | ✅ | ❌ blocked | Use concrete model |
| `or '''fallback'''` syntax | ✅ | ❌ changed | — | `||| fallback` operator |

---

## Common Patterns

### Auto-send on state change

```python
class MyModel(models.Model):
    _name = 'my.model'
    _inherit = ['mail.thread']

    state = fields.Selection([('draft', 'Draft'), ('confirmed', 'Confirmed')])

    def action_confirm(self):
        self.state = 'confirmed'
        template = self.env.ref('my_module.email_template_confirmed')
        for rec in self:
            template.send_mail(rec.id, force_send=True)
        return True
```

### Render template field for preview

```python
# Render subject for a specific record (without sending)
rendered = template._render_field(
    'subject',
    [self.id],
    compute_lang=True,
)
# rendered = {self.id: 'Rendered Subject Text'}
```

### Dynamic language selection

```python
# Template lang field (Jinja2 expression):
# "{{ object.partner_id.lang }}"    ← use partner's language
# "{{ user.lang }}"                 ← use sender's language
# "en_US"                           ← hardcode
# ""                                ← use company default (v19: main partner's lang)
```
