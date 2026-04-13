---
name: odoo-i18n
description: >
  Expert Odoo internationalization, translation, and localization for versions 17.0, 18.0, 19.0.
  Covers _() and _lt() usage, .po/.pot files, translate=True fields, --i18n-export/import CLI,
  website translation, RTL languages, and common pitfalls. Use when user mentions translations,
  .po files, _lt(), ir.translation, multi-language, RTL, Arabic, export/import translations,
  missing translation, translatable fields, or invokes /odoo-i18n.
---

Expert Odoo i18n engineer. Target: 17/18/19 CE+EE.

**Always ask**: new translation, fixing broken one, or setting up a language? The fix differs.

---

## 1. Import Conventions (source-verified v17/v18/v19)

```python
# Standard _ comes from odoo directly
from odoo import _, api, fields, models

# For lazy translation (class-level strings): import _lt separately
# _lt = LazyGettext — source: odoo/tools/translate.py:732
from odoo.tools.translate import _lt

# Usage summary:
# _()   → translates immediately using current request language
# _lt() → translates lazily — use for module-level / class-level constants
```

---

## 2. _() — Immediate Translation

Use `_()` everywhere inside methods (actions, compute, onchange, wizard logic).

```python
from odoo import _, models, fields
from odoo.exceptions import UserError, ValidationError


class MyModel(models.Model):
    _name = 'my.model'

    def action_confirm(self):
        for rec in self:
            if not rec.partner_id:
                raise UserError(_('Please select a partner before confirming.'))
        self.write({'state': 'confirmed'})

    @api.constrains('date_start', 'date_end')
    def _check_dates(self):
        for rec in self:
            if rec.date_start and rec.date_end and rec.date_start > rec.date_end:
                raise ValidationError(
                    _('End date %(end)s must be after start date %(start)s.',
                      end=rec.date_end, start=rec.date_start)
                )
```

### String formatting rules

```python
# ✅ GOOD — use named placeholders (visible to translator)
_('Hello, %(name)s!', name=partner.name)
_('Invoice %(ref)s has %(count)d lines.', ref=inv.name, count=len(inv.line_ids))

# ❌ BAD — positional % — context lost for translator
_('Hello, %s!') % partner.name

# ❌ BAD — f-string inside _() — not extracted by gettext
_(f'Hello, {partner.name}!')

# ❌ BAD — variable string — cannot be extracted statically
msg = 'Hello'
_(msg)
```

---

## 3. _lt() — Lazy Translation

Use `_lt()` for strings defined at class or module level (outside methods), where the language context may not be available yet.

```python
from odoo.tools.translate import _lt
from odoo import models, fields


class MyModel(models.Model):
    _name = 'my.model'
    _description = 'My Model'  # ← always a plain string, not translated

    STATE_LABELS = {
        'draft': _lt('Draft'),
        'confirmed': _lt('Confirmed'),
        'done': _lt('Done'),
    }

    state = fields.Selection([
        ('draft', 'Draft'),         # Selection labels use plain strings here
        ('confirmed', 'Confirmed'), # Odoo extracts them automatically
        ('done', 'Done'),
    ])

    def get_state_label(self):
        return str(self.STATE_LABELS[self.state])   # call str() to force resolution
```

---

## 4. Translatable Fields

### Model fields with translate=True

```python
class ProductTemplate(models.Model):
    _inherit = 'product.template'

    marketing_tagline = fields.Char(string='Marketing Tagline', translate=True)
    long_description = fields.Html(string='Long Description', translate=True)

    # Fields with translate=True:
    # - Store one value per language in the DB (ir.translation / jsonb in v17+)
    # - Accessible via: rec.with_context(lang='fr_FR').marketing_tagline
    # - Exported with --i18n-export and importable via .po files
```

### Read in specific language

```python
# Read the field value in French
fr_product = product.with_context(lang='fr_FR')
print(fr_product.name)   # French name if translated

# Write translation
product.with_context(lang='fr_FR').write({'name': 'Produit Example'})
```

---

## 5. Module .po File Structure

Every translatable module should have:

```
my_module/
├── i18n/
│   ├── my_module.pot      ← master template (auto-generated)
│   ├── fr.po              ← French translation
│   ├── ar.po              ← Arabic (RTL)
│   └── es.po              ← Spanish
```

### .pot / .po file header

```po
# Translation of my_module in French
# Copyright (C) 2024 Your Company
msgid ""
msgstr ""
"Project-Id-Version: Odoo 18.0\n"
"PO-Revision-Date: 2024-01-01 00:00+0000\n"
"Last-Translator: \n"
"Language-Team: French\n"
"Language: fr\n"
"MIME-Version: 1.0\n"
"Content-Type: text/plain; charset=UTF-8\n"
"Content-Transfer-Encoding: 8bit\n"

msgid "Please select a partner before confirming."
msgstr "Veuillez sélectionner un partenaire avant de confirmer."

msgid "End date %(end)s must be after start date %(start)s."
msgstr "La date de fin %(end)s doit être après la date de début %(start)s."
```

---

## 6. Export & Import Translations

### Export .pot template (generate from source)

```bash
# Export all translatable strings from a module to .pot
./odoo-bin --i18n-export=/tmp/my_module.pot \
  --modules=my_module \
  --language=en \
  -d yourdb \
  --stop-after-init

# Export as .po for a specific language (existing translations)
./odoo-bin --i18n-export=/tmp/my_module_fr.po \
  --modules=my_module \
  --language=fr_FR \
  -d yourdb \
  --stop-after-init
```

### Import translations

```bash
# Import a .po file into the database
./odoo-bin --i18n-import=/path/to/fr.po \
  --language=fr_FR \
  --modules=my_module \
  -d yourdb \
  --stop-after-init
```

### Via UI (Settings > Translations)

```
Settings → Translations → Import Translation → upload .po file
Settings → Translations → Export Translation → download .po file
Settings → Translations → Languages → install language packs
```

### Auto-load at install

Place `.po` file in `i18n/` — Odoo loads it automatically when language is installed/module updated.

---

## 7. QWeb / XML Translations

```xml
<!-- Translated automatically if in <t t-esc> or text content -->
<p>This text is extracted.</p>

<!-- Use t-out with translation context -->
<t t-out="record.name"/>

<!-- Explicit translate attribute in arch -->
<field name="arch" type="xml">
    <form>
        <button string="Confirm" type="object" name="action_confirm"/>
        <!-- string= attribute is always extracted and translated -->
    </form>
</field>
```

### Email templates (mail.template)

```xml
<record id="email_template_example" model="mail.template">
    <field name="name">Example Email</field>
    <field name="subject">Invoice {{ object.name }}</field>
    <!-- body_html is translated — add per-language versions via UI -->
    <field name="body_html" type="html">
        <p>Dear <t t-out="object.partner_id.name"/>,</p>
        <p>Your invoice is ready.</p>
    </field>
</record>
```

---

## 8. RTL Language Support (Arabic, Hebrew, Farsi)

```python
# Detect RTL in Python
lang = self.env.lang or self.env.context.get('lang', 'en_US')
lang_record = self.env['res.lang'].search([('code', '=', lang)], limit=1)
is_rtl = lang_record.direction == 'rtl'
```

```xml
<!-- QWeb: apply RTL direction dynamically -->
<div t-attf-style="direction: {{ 'rtl' if lang_direction == 'rtl' else 'ltr' }}">
    ...
</div>

<!-- CSS for RTL in QWeb reports -->
<style>
    /* RTL support in PDF reports */
    body[dir="rtl"] .text-right { text-align: left !important; }
    body[dir="rtl"] .text-left  { text-align: right !important; }
</style>
```

---

## 9. Website / Portal Translations

```python
# Controller: pass lang explicitly for website pages
class MyController(http.Controller):
    @http.route('/my-page', auth='public', website=True)
    def my_page(self, **kwargs):
        return request.render('my_module.my_page_template', {})
        # Website module sets lang from URL (/fr/my-page) automatically
```

```xml
<!-- Website template — wrap translatable blocks -->
<template id="my_page_template">
    <t t-call="website.layout">
        <div id="wrap">
            <h1>Welcome</h1>  <!-- extracted as-is -->
            <p>
                Contact us at
                <a t-att-href="'mailto:' + company.email">
                    <t t-out="company.email"/>
                </a>
            </p>
        </div>
    </t>
</template>
```

---

## 10. Common Pitfalls

### Problem: translation not showing

```
1. Language not installed → Settings > Languages > install
2. Module not updated after adding .po → ./odoo-bin -u my_module
3. String not matching exactly (whitespace, punctuation) → check .po msgid
4. Used f-string or % format in _() → not extracted, fix the string
5. Caching → restart Odoo after import
```

### Problem: `_lt` not resolving

```python
# ✅ Force resolution to str when used as a regular string
label = str(_lt('My Label'))

# ✅ _lt resolves automatically when used in UserError/ValidationError
raise UserError(_lt('Something went wrong.'))   # OK — resolved on display
```

### Problem: `translate=True` field not showing translated value

```python
# ❌ BAD — no lang context
value = record.name

# ✅ GOOD — explicit lang context
value = record.with_context(lang=self.env.user.lang).name
```

---

## 11. Version Notes

| Feature | v17 | v18 | v19 |
|---------|-----|-----|-----|
| `_()` from `odoo` | ✅ | ✅ | ✅ |
| `_lt` in `odoo.tools.translate` | ✅ | ✅ | ✅ |
| `translate=True` on fields | ✅ | ✅ | ✅ |
| `--i18n-export` / `--i18n-import` | ✅ | ✅ | ✅ |
| `ir.translation` model | ✅ (legacy) | ✅ (legacy) | ✅ (legacy) |
| Translation stored in jsonb (field-level) | ✅ | ✅ | ✅ |
