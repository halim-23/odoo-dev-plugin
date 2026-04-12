---
name: odoo-portal
description: >
  Odoo portal development for versions 17.0, 18.0, 19.0 (Community & Enterprise).
  Covers CustomerPortal controller, portal.mixin, route patterns (list/detail/JSON/sitemap),
  pagination helper, _prepare_home_portal_values, _document_check_access, portal QWeb templates
  (portal_layout, portal_record_layout, portal_sidebar, breadcrumbs, pager, message_thread),
  chatter in portal (QWeb v17 → OWL v18/v19), access token / _sign_token / portal sharing,
  attachment upload (v17 /portal/attachment/add → v18+ mail module),
  modular controller architecture (v18: mail.py/thread.py/attachment.py/web.py/utils.py),
  portal.utils validate functions (v18+), address management overhaul (v19),
  public.interactions registry usage (v19), Domain class in portal_thread.py (v19),
  SignatureForm OWL component (v19), portal counter session caching (v18/v19),
  EE portal features (helpdesk, sign, field_service), security groups (group_portal, group_public),
  ir.model.access for portal users, full version diff table.
  Use when user asks about portal pages, portal routes, portal templates, portal chatter,
  portal sharing, portal access token, or invokes /odoo-portal.
---

# Odoo Portal Development

Source-verified against v17.0, v18.0, v19.0 Community & Enterprise.

---

## Version Feature Matrix

| Feature | v17 | v18 | v19 |
|---|---|---|---|
| `attachment_add` route | ✓ `/portal/attachment/add` | ✗ Removed | ✗ Removed |
| Attachment handled by | portal module | mail module | mail module |
| Chatter tech | QWeb templates | OWL components | OWL components |
| `check_access_rights/rule` | ✓ two calls | ✗ → `check_access()` | `check_access()` |
| MANDATORY_BILLING_FIELDS | class constant | method `_get_mandatory_fields()` | method |
| Portal counter caching | ✗ | ✓ session cache | ✓ session cache |
| Message editing by portal | ✗ | ✓ `_is_editable_in_portal()` | ✓ |
| Message reactions | ✗ | ✓ | ✓ |
| Modular controller files | 1 (portal.py) | 6 files | 6+ files |
| `utils.py` auth helpers | ✗ | ✓ | ✓ |
| `public.interactions` registry | ✗ | ✗ | ✓ |
| Address management routes | basic `/my/account` | basic | full `/my/addresses` CRUD |
| `Domain` class in portal | ✗ | ✗ | ✓ `portal_thread.py` |
| `SignatureForm` OWL | ✗ | ✗ | ✓ |
| Triple auth (hash/token/signup) | partial | ✓ | ✓ |
| `portal.assets_chatter` bundle | ✗ | ✓ | ✓ |
| `share_token` param on `_get_share_url` | ✓ | ✓ | ✓ (explicit default=True) |

---

## 1. portal.mixin — Abstract Model

```python
# addons/portal/models/portal_mixin.py
class PortalMixin(models.AbstractModel):
    _name = "portal.mixin"
    _description = 'Portal Mixin'

    access_url = fields.Char('Portal Access URL', compute='_compute_access_url')
    access_token = fields.Char('Security Token', copy=False)  # UUID
    access_warning = fields.Text("Access warning", compute="_compute_access_warning")
```

### Adding to your model

```python
class MyModel(models.Model):
    _name = 'my.model'
    _inherit = ['portal.mixin', 'mail.thread', 'mail.activity.mixin']

    def _compute_access_url(self):
        for rec in self:
            rec.access_url = '/my/mymodel/%s' % rec.id
```

### Key mixin methods

| Method | Purpose | Notes |
|---|---|---|
| `_portal_ensure_token()` | Generate UUID token if missing | Unchanged v17→v19 |
| `_get_share_url(redirect, signup_partner, pid, share_token)` | Build shareable URL | `share_token=True` default |
| `get_portal_url(suffix, report_type, download, query_string, anchor)` | URL builder for templates | Unchanged |
| `_compute_access_url()` | Override in your model | Returns `#` by default |
| `_get_access_action(access_uid, force_website)` | Redirect portal users | Override to set portal URL |

### `_get_share_url()` signature

```python
# v17 / v18 / v19 — same signature
def _get_share_url(self, redirect=False, signup_partner=False, pid=None, share_token=True):
    """
    redirect=True  → /mail/view?model=...&res_id=... (backend redirect URL)
    redirect=False → direct portal URL with access_token
    pid            → adds pid + hash for portal chatter auth
    share_token    → v19 adds explicit param; when False, omit access_token
    """
```

### Access check — VERSION BREAKING CHANGE

```python
# v17: TWO separate calls
record.check_access_rights('read')
record.check_access_rule('read')

# v18 / v19: ONE unified call
record.check_access('read')
```

---

## 2. CustomerPortal Controller

```python
# Inherit in your module
from odoo.addons.portal.controllers import portal

class MyPortal(portal.CustomerPortal):
    pass
```

### Class constants → methods (v17 → v18)

```python
# v17: class-level constants
class CustomerPortal(Controller):
    MANDATORY_BILLING_FIELDS = ["name", "phone", "email", "street", "city", "country_id"]
    OPTIONAL_BILLING_FIELDS = ["zipcode", "state_id", "vat", "company_name"]
    _items_per_page = 80

# v18 / v19: methods (override-friendly)
def _get_mandatory_fields(self):
    return ["name", "phone", "email", "street", "city", "country_id"]

def _get_optional_fields(self):
    return ["street2", "zipcode", "state_id", "vat", "company_name"]
```

### `_prepare_home_portal_values(counters)`

Override to add counters to portal home page badges:

```python
def _prepare_home_portal_values(self, counters):
    values = super()._prepare_home_portal_values(counters)
    partner = request.env.user.partner_id
    MyModel = request.env['my.model']
    if 'mymodel_count' in counters:
        values['mymodel_count'] = MyModel.search_count(
            [('partner_id', '=', partner.id)]
        )
    return values
```

### `_prepare_portal_layout_values()`

Returns common values for all portal pages. Does NOT include record counts.

```python
# Returns:
{
    'sales_user': sales_user_sudo,  # Assigned sales rep
    'page_name': 'home',
}
```

### `/my/counters` route — v18 adds session caching

```python
# v17
@route(['/my/counters'], type='json', auth="user", website=True)
def counters(self, counters, **kw):
    return self._prepare_home_portal_values(counters)

# v18 / v19: adds readonly=True + session cache
@route(['/my/counters'], type='json', auth="user", website=True, readonly=True)
def counters(self, counters, **kw):
    cache = (request.session.portal_counters or {}).copy()
    res = self._prepare_home_portal_values(counters)
    cache.update({k: bool(v) for k, v in res.items() if k.endswith('_count')})
    if cache != request.session.portal_counters:
        request.session.portal_counters = cache
    return res
```

### `/my/home` route — v18 change

```python
# v17
def home(self, **kw):
    values = self._prepare_portal_layout_values()
    return request.render("portal.portal_my_home", values)

# v18 / v19: also calls _prepare_home_portal_values([]) eagerly
def home(self, **kw):
    values = self._prepare_portal_layout_values()
    values.update(self._prepare_home_portal_values([]))
    return request.render("portal.portal_my_home", values)
```

### `_get_page_view_values()` (v17 only)

v17 has `_get_page_view_values(document, access_token, values, session_history, no_breadcrumbs)`.
Prepares values for single record view: sets `object`, `token`, `pid`, `hash`, prev/next.

v18/v19: replaced by `_get_thread_with_access()` on the model side + inline controller logic.

---

## 3. Route Patterns

### List route (auth="user")

```python
@http.route(
    ['/my/myitems', '/my/myitems/page/<int:page>'],
    type='http', auth="user", website=True
)
def portal_my_items(self, page=1, date_begin=None, date_end=None,
                    sortby=None, filterby='all', **kw):
    MyModel = request.env['my.model']
    partner = request.env.user.partner_id

    domain = [('partner_id', '=', partner.id)]

    searchbar_sortings = {
        'date': {'label': _('Newest'), 'order': 'create_date desc'},
        'name': {'label': _('Name'), 'order': 'name'},
    }
    if not sortby or sortby not in searchbar_sortings:
        sortby = 'date'
    order = searchbar_sortings[sortby]['order']

    total = MyModel.search_count(domain)
    pager_vals = portal_pager(
        url='/my/myitems',
        total=total, page=page,
        step=self._items_per_page,
        url_args={'sortby': sortby, 'filterby': filterby},
    )
    records = MyModel.search(
        domain, order=order,
        limit=self._items_per_page,
        offset=pager_vals['offset'],
    )
    return request.render('my_module.portal_my_items', {
        'myitems': records,
        'page_name': 'myitem',
        'pager': pager_vals,
        'default_url': '/my/myitems',
        'searchbar_sortings': searchbar_sortings,
        'sortby': sortby,
    })
```

### Detail route (auth="public" + access_token)

```python
@http.route(
    ['/my/myitem/<int:record_id>'],
    type='http', auth="public", website=True
)
def portal_my_item(self, record_id, access_token=None, **kw):
    try:
        record_sudo = self._document_check_access(
            'my.model', record_id, access_token=access_token
        )
    except (AccessError, MissingError):
        return request.redirect('/my')

    # Build pid + hash for portal chatter (v17/v18)
    values = {
        'record': record_sudo,
        'page_name': 'myitem',
        'token': access_token,
        'pid': None,
        'hash': None,
    }
    if request.env.user != request.env.ref('base.public_user'):
        partner = request.env.user.partner_id
        values.update({
            'pid': partner.id,
            'hash': record_sudo._sign_token(partner.id),
        })

    return request.render('my_module.portal_my_item_page', values)
```

### JSON action route

```python
@http.route(
    ['/my/myitem/<int:record_id>/confirm'],
    type='json', auth="public", website=True
)
def portal_item_confirm(self, record_id, access_token=None, **kw):
    record_sudo = self._document_check_access(
        'my.model', record_id, access_token=access_token
    )
    record_sudo.action_confirm()
    return {'success': True}
```

### Sitemap route

```python
def _sitemap_my_items(env, rule, qs):
    """Yield sitemap entries for published items."""
    records = env['my.model'].search([('website_published', '=', True)])
    for r in records:
        yield {'loc': r.access_url}

@http.route('/my/myitem/<int:record_id>', type='http', auth='public',
            website=True, sitemap=_sitemap_my_items)
def portal_my_item(self, record_id, **kw):
    ...
```

---

## 4. Document Access Check

### `_document_check_access()` — v17 / v18 / v19

```python
def _document_check_access(self, model_name, document_id, access_token=None):
    """Verify access to a record, fallback to access_token.

    Returns record with SUPERUSER context on success.
    Raises AccessError or MissingError on failure.
    """
    document = request.env[model_name].browse([document_id])
    document_sudo = document.with_user(SUPERUSER_ID).exists()
    if not document_sudo:
        raise MissingError(_("This document does not exist."))
    try:
        # v17: two separate calls
        document.check_access_rights('read')
        document.check_access_rule('read')
        # v18/v19: single call
        # document.check_access('read')
    except AccessError:
        if not access_token or not document_sudo.access_token or \
           not consteq(document_sudo.access_token, access_token):
            raise
    return document_sudo
```

---

## 5. Pagination

```python
from odoo.addons.portal.controllers.portal import pager as portal_pager

pager_vals = portal_pager(
    url='/my/myitems',           # Base URL (no page suffix)
    total=total_count,            # Total matching records
    page=page,                    # Current page (int, from route)
    step=self._items_per_page,    # Records per page (default 80)
    scope=5,                      # Pages to show in UI
    url_args={'sortby': sortby},  # Extra query params preserved in links
)

# Database query
records = Model.search(domain, order=order,
                       limit=self._items_per_page,
                       offset=pager_vals['offset'])

# Template: pass pager to values
values['pager'] = pager_vals
# In QWeb: <t t-call="portal.pager"/>
```

---

## 6. QWeb Templates Reference

### Template hierarchy

```
portal.frontend_layout         (website wrapper)
  └── portal.portal_layout     (portal content wrapper)
        ├── portal.portal_breadcrumbs
        ├── portal.record_pager        (prev/next)
        ├── portal.portal_sidebar      (sidebar layout)
        │     └── portal.side_content  (avatar + contact)
        └── portal.portal_record_layout (card wrapper)
```

### `portal.portal_layout` — page wrapper

```xml
<t t-call="portal.portal_layout">
    <t t-set="page_name" t-value="'myitem'"/>
    <t t-set="no_breadcrumbs" t-value="False"/>

    <!-- breadcrumb items injected via t-call param -->
    <t t-set="breadcrumbs_searchbar" t-value="True"/>

    <!-- Your page content here -->
    <div class="container o_portal_sidebar">
        ...
    </div>
</t>
```

**Variables accepted by `portal_layout`:**

| Variable | Type | Purpose |
|---|---|---|
| `page_name` | str | Breadcrumb page label |
| `no_breadcrumbs` | bool | Hide breadcrumbs |
| `breadcrumbs_searchbar` | bool | Show breadcrumbs instead of title |
| `prev_record` | url | Previous record URL |
| `next_record` | url | Next record URL |

### `portal.portal_record_layout` — card wrapper

```xml
<t t-call="portal.portal_record_layout">
    <t t-set="card_header">
        <h5 class="mb-0">Order #<t t-out="record.name"/></h5>
    </t>
    <t t-set="card_body">
        <!-- record body content -->
    </t>
</t>
```

### `portal.pager` — pagination

```xml
<div class="o_portal_pager text-center">
    <t t-if="pager['page_count'] > 1" t-call="portal.pager"/>
</div>
```

### `portal.portal_breadcrumbs` — custom breadcrumbs

```xml
<t t-call="portal.portal_breadcrumbs">
    <t t-set="additional_title">
        <li class="breadcrumb-item">
            <a href="/my/myitems">My Items</a>
        </li>
        <li class="breadcrumb-item active">
            <t t-out="record.name"/>
        </li>
    </t>
</t>
```

### `portal.portal_sidebar` — sidebar layout

```xml
<t t-call="portal.portal_sidebar">
    <t t-set="o_portal_fullwidth_alert">
        <!-- Optional full-width alert at top -->
    </t>
    <!-- Sidebar + main content provided by portal.portal_layout -->
</t>
```

---

## 7. Chatter in Portal

### v17 — QWeb `message_thread` template

```python
# Controller: prepare pid + hash
partner = request.env.user.partner_id
values = {
    'object': record_sudo,
    'token': access_token,
    'pid': partner.id if not request.env.user._is_public() else None,
    'hash': record_sudo._sign_token(partner.id) if not request.env.user._is_public() else None,
    'disable_composer': False,
    'message_per_page': 10,
}
```

```xml
<!-- v17 template: renders QWeb-based chatter -->
<div id="communication" class="mt-4">
    <h3>Communication</h3>
    <t t-call="portal.message_thread">
        <t t-set="object" t-value="record"/>
        <t t-set="token" t-value="token"/>
        <t t-set="pid" t-value="pid"/>
        <t t-set="hash" t-value="hash"/>
        <t t-set="disable_composer" t-value="False"/>
        <t t-set="message_per_page" t-value="10"/>
    </t>
</div>
```

### v18 / v19 — OWL `PortalChatter` component

The `portal.message_thread` template still exists but now renders a **data container** that boots the OWL chatter:

```xml
<!-- v18/v19: template outputs a div with data-* attrs -->
<template id="message_thread">
    <div id="discussion" data-anchor="true"
         class="o_portal_chatter o_not_editable p-0"
         t-att-data-token="token"
         t-att-data-res_model="object._name"
         t-att-data-pid="pid"
         t-att-data-hash="hash"
         t-att-data-res_id="object.id"
         t-att-data-pager_step="message_per_page or 10"
         t-att-data-allow_composer="'0' if disable_composer else '1'"
         t-att-data-two_columns="'true' if two_columns else 'false'">
    </div>
</template>
```

OWL `PortalChatter` component reads `data-*` attributes and mounts `<Chatter>`.

**New v18 endpoint for chatter init:**
```
POST /portal/chatter_init   (type=json, auth=public)
POST /mail/chatter_fetch     (type=json, auth=public)
```

**New v18 chatter features:**
- Portal users can **edit** their own messages (`_is_editable_in_portal()`)
- Portal users can **react** to messages
- Portal-specific avatar endpoint: `/mail/avatar/mail.message/<id>/author_avatar/<size>`

---

## 8. Access Token & Portal Signing

### `_portal_ensure_token()` — generate UUID token

```python
# Model method — unchanged v17→v19
def _portal_ensure_token(self):
    if not self.access_token:
        self.sudo().write({'access_token': str(uuid.uuid4())})
    return self.access_token
```

### `_sign_token(pid)` — HMAC for portal chatter auth

```python
# mail_thread.py — unchanged v17→v19
def _sign_token(self, pid):
    secret = self.env["ir.config_parameter"].sudo().get_param("database.secret")
    token = (self.env.cr.dbname, self[self._mail_post_token_field], pid)
    return hmac.new(
        secret.encode('utf-8'),
        repr(token).encode('utf-8'),
        hashlib.sha256
    ).hexdigest()
```

### v18+ `utils.py` — portal validation helpers (NEW)

```python
# portal/utils.py (v18+)
from odoo.addons.portal.utils import (
    validate_thread_with_hash_pid,
    validate_thread_with_token,
    get_portal_partner,
)

# Validate email recipient via HMAC
validate_thread_with_hash_pid(thread, _hash, pid)  # bool

# Validate direct access_token
validate_thread_with_token(thread, token)  # bool

# Resolve portal partner from any auth method
partner = get_portal_partner(thread, _hash=hash, pid=pid, token=token)
```

### `_mail_post_token_field` — configure token field

```python
class MyModel(models.Model):
    _inherit = 'my.model'
    # By default: 'access_token'
    _mail_post_token_field = 'access_token'
```

### Triple authentication (v18+)

| Method | When Used | Params |
|---|---|---|
| Hash/PID | Email recipients (HMAC signed) | `?pid=X&hash=HMAC` |
| Direct token | Shared links | `?access_token=UUID` |
| Signup token | Partner signup flow | `?token=signup_token` |

---

## 9. Attachment Upload

### v17 — `/portal/attachment/add` (in portal module)

```python
# v17 controller — portal/controllers/portal.py
@http.route('/portal/attachment/add', type='http', auth='public',
            methods=['POST'], website=True)
def attachment_add(self, name, file, res_model, res_id, access_token=None, **kwargs):
    self._document_check_access(res_model, int(res_id), access_token=access_token)
    IrAttachment = request.env['ir.attachment']
    if not request.env.user._is_internal():
        IrAttachment = IrAttachment.sudo()
    attachment = IrAttachment.create({
        'name': name,
        'datas': base64.b64encode(file.read()),
        'res_model': 'mail.compose.message',  # pending state
        'res_id': 0,
        'access_token': IrAttachment._generate_access_token(),
    })
    return request.make_response(
        data=json.dumps(attachment.read(['id', 'name', 'mimetype', 'file_size', 'access_token'])[0]),
        headers=[('Content-Type', 'application/json')]
    )
```

### v18 / v19 — moved to mail module

```
POST /mail/attachment/upload   ← use this in v18/v19
```

Handled by `mail.controllers.attachment.AttachmentController`. Portal module has
`PortalAttachmentController(AttachmentController)` that overrides `_is_allowed_to_delete()`.

### Attachment remove — v17

```python
@http.route('/portal/attachment/remove', type='json', auth='public')
def attachment_remove(self, attachment_id, access_token=None):
    attachment_sudo = self._document_check_access(
        'ir.attachment', int(attachment_id), access_token=access_token
    )
    if attachment_sudo.res_model != 'mail.compose.message' or attachment_sudo.res_id != 0:
        raise UserError(_("Only pending attachments can be removed."))
    return attachment_sudo.unlink()
```

---

## 10. Modular Controller Structure (v18+)

v18 split the monolithic `portal.py` into:

| File | Class | Key Routes/Methods |
|---|---|---|
| `portal.py` | `CustomerPortal` | `/my`, `/my/home`, `/my/account`, `/my/security` |
| `mail.py` | `PortalChatter`, `MailController` | `/portal/chatter_init`, `/mail/chatter_fetch` |
| `thread.py` | `ThreadController` | `_prepare_post_data`, `_is_message_editable` |
| `attachment.py` | `PortalAttachmentController` | `_is_allowed_to_delete` |
| `message_reaction.py` | `PortalMessageReactionController` | `_get_reaction_author` |
| `web.py` | `Home` | Redirect non-internal to `/my` |

v19 adds `portal_thread.py`:

| File | Class | Key Routes |
|---|---|---|
| `portal_thread.py` | `PortalThread` | `/mail/thread/messages` (with `Domain` class) |

---

## 11. Address Management (v19 — Major Overhaul)

v19 adds a full address management system at `/my/addresses`:

### New routes (v19)

```python
@http.route(['/my/addresses'], type='http', auth="user", website=True)
def my_addresses(self, **kw): ...

@http.route(['/my/address'], type='http', auth="user", website=True, methods=['GET'])
def portal_address(self, **kw): ...  # Show form

@http.route(['/my/address/submit'], type='http', auth="user", website=True, methods=['POST'])
def portal_address_submit(self, **kw): ...  # Create/update

@http.route(['/my/address/archive'], type='json', auth="user", website=True)
def address_archive(self, **kw): ...  # Archive address
```

### New helper methods (v19)

```python
def _parse_form_data(self, data):          # Convert form POST → partner fields
def _validate_address_values(self, vals):  # Field-level validation per country
def _complete_address_values(self, vals):  # Fill language, company, type
def _are_same_addresses(self, vals, partner): ...
def _handle_extra_form_data(self, vals, **kw): ...  # Hook for extra fields
def _prepare_address_form_values(self, ...):  # Prepare form template values
```

### New templates (v19)

| Template | Purpose |
|---|---|
| `portal.my_addresses` | Address list page |
| `portal.address_card` | Single address card widget |
| `portal.address_form_fields` | Address form fields |
| `portal.address_footer` | Form footer buttons |
| `portal.address_management` | Address management container |

---

## 12. public.interactions Registry (v19)

v19 portal replaces inline JS with `public.interactions` registry components:

```javascript
// v19 pattern — interactions/*.js
import { Interaction } from "@web/public/interaction";
import { registry } from "@web/core/registry";

class PortalDetails extends Interaction {
    static selector = ".o_portal_details";  // CSS selector for mount point

    dynamicContent = {
        ".o_portal_country_select": {
            "t-on-change": this.onChangeCountry,
        },
    };

    async onChangeCountry() {
        const countryId = this.el.querySelector(".o_portal_country_select").value;
        const result = await this.waitFor(
            this.rpc(`/my/address/country_info/${countryId}`)
        );
        // Update state dropdown visibility
    }
}

registry.category("public.interactions").add("portal.portal_details", PortalDetails);
```

### Portal interactions registered in v19

| Class | Selector | Purpose |
|---|---|---|
| `PortalDetails` | `.o_portal_details` | Country/state cascade on account form |
| `PortalComposer` | `.o_portal_chatter_composer` | File upload, message post |
| `CustomerAddress` | `.o_customer_address_fill` | Address form country/state cascade |
| `AddressCard` | `.o_address_card` | Address card edit/remove |
| `PortalHomeCounters` | portal home | Badge counter fetch |
| `PortalSearchPanel` | `.o_portal_search_panel` | Search/filter panel |
| `PortalSecurity` | security page | Password change, account deactivation |
| `Sidebar` | sidebar | Timeago labels, print |

---

## 13. Domain Class in Portal (v19)

`portal_thread.py` (v19) uses the new `Domain` class:

```python
# v19 — portal/controllers/portal_thread.py
from odoo.fields import Domain

domain = (
    Domain(self._setup_portal_message_fetch_extra_domain(kw))
    & Domain(field.get_comodel_domain(model))
    & Domain("res_id", "=", thread_id)
    & Domain("subtype_id", "=", request.env.ref("mail.mt_comment").id)
    & self._get_non_empty_message_domain()
)

def _setup_portal_message_fetch_extra_domain(self, kw):
    return Domain.TRUE  # Subclasses override to add extra filters

def _get_non_empty_message_domain(self):
    return Domain("body", "!=", "") | Domain("attachment_ids", "!=", False)
```

---

## 14. Security — Portal User Access

### Security groups

```
base.group_public    → unauthenticated visitors (auth="public")
base.group_portal    → portal users (auth="user" — logged-in portal)
base.group_user      → internal users
```

### CSV access rights for portal users

```csv
# ir.model.access.csv
id,name,model_id:id,group_id:id,perm_read,perm_write,perm_create,perm_unlink
access_my_model_portal,my.model portal,model_my_model,base.group_portal,1,0,0,0
```

### Record rules for portal users

```xml
<record id="rule_my_model_portal" model="ir.rule">
    <field name="name">My Model: portal sees own records</field>
    <field name="model_id" ref="model_my_model"/>
    <field name="domain_force">[('partner_id', '=', user.partner_id.id)]</field>
    <field name="groups" eval="[(4, ref('base.group_portal'))]"/>
</record>
```

### Route auth types

| `auth=` | Who can access | Use for |
|---|---|---|
| `"user"` | Logged-in users (portal + internal) | List pages |
| `"public"` | Anyone (including anonymous) | Detail pages with `access_token` |
| `"none"` | No session required | Low-level endpoints |

---

## 15. Full Portal Page — Complete Pattern

### Controller

```python
# controllers/portal.py
from odoo.addons.portal.controllers.portal import CustomerPortal, pager as portal_pager
from odoo import http
from odoo.http import request
from odoo.exceptions import AccessError, MissingError

class MyPortal(CustomerPortal):

    def _prepare_home_portal_values(self, counters):
        values = super()._prepare_home_portal_values(counters)
        partner = request.env.user.partner_id
        if 'myitem_count' in counters:
            values['myitem_count'] = request.env['my.item'].search_count(
                [('partner_id', '=', partner.id)]
            )
        return values

    @http.route(['/my/myitems', '/my/myitems/page/<int:page>'],
                type='http', auth="user", website=True)
    def portal_my_items(self, page=1, sortby='date', **kw):
        partner = request.env.user.partner_id
        domain = [('partner_id', '=', partner.id)]
        total = request.env['my.item'].search_count(domain)
        pager_vals = portal_pager(
            url='/my/myitems', total=total, page=page,
            step=self._items_per_page, url_args={'sortby': sortby},
        )
        records = request.env['my.item'].search(
            domain, limit=self._items_per_page, offset=pager_vals['offset'],
            order='create_date desc',
        )
        return request.render('my_module.portal_my_items', {
            'myitems': records,
            'pager': pager_vals,
            'page_name': 'myitem',
        })

    @http.route(['/my/myitem/<int:item_id>'],
                type='http', auth="public", website=True)
    def portal_my_item(self, item_id, access_token=None, **kw):
        try:
            item_sudo = self._document_check_access(
                'my.item', item_id, access_token=access_token
            )
        except (AccessError, MissingError):
            return request.redirect('/my')

        values = {
            'item': item_sudo,
            'object': item_sudo,      # required by portal.message_thread
            'token': access_token,
            'page_name': 'myitem',
            'pid': None,
            'hash': None,
        }
        if not request.env.user._is_public():
            pid = request.env.user.partner_id.id
            values.update({'pid': pid, 'hash': item_sudo._sign_token(pid)})

        return request.render('my_module.portal_my_item_page', values)
```

### Model

```python
# models/my_item.py
class MyItem(models.Model):
    _name = 'my.item'
    _inherit = ['portal.mixin', 'mail.thread', 'mail.activity.mixin']
    _description = 'My Item'

    name = fields.Char(required=True)
    partner_id = fields.Many2one('res.partner', required=True)

    def _compute_access_url(self):
        for rec in self:
            rec.access_url = '/my/myitem/%s' % rec.id
```

### List template

```xml
<!-- views/portal_templates.xml -->
<template id="portal_my_items" name="My Items">
    <t t-call="portal.portal_layout">
        <t t-set="breadcrumbs_searchbar" t-value="True"/>

        <t t-call="portal.portal_searchbar">
            <t t-set="title">My Items</t>
        </t>

        <t t-if="not myitems">
            <div class="alert alert-warning">No items found.</div>
        </t>
        <t t-if="myitems">
            <div class="card">
                <div class="table-responsive">
                    <table class="table table-hover o_portal_my_doc_table">
                        <thead>
                            <tr>
                                <th>Name</th>
                                <th>Date</th>
                            </tr>
                        </thead>
                        <tbody>
                            <t t-foreach="myitems" t-as="item">
                                <tr>
                                    <td>
                                        <a t-att-href="item.get_portal_url()">
                                            <t t-out="item.name"/>
                                        </a>
                                    </td>
                                    <td>
                                        <span t-field="item.create_date"
                                              t-options='{"widget": "date"}'/>
                                    </td>
                                </tr>
                            </t>
                        </tbody>
                    </table>
                </div>
            </div>
            <div class="o_portal_pager text-center">
                <t t-call="portal.pager"/>
            </div>
        </t>
    </t>
</template>
```

### Detail template

```xml
<template id="portal_my_item_page" name="My Item">
    <t t-call="portal.portal_layout">
        <t t-set="page_name" t-value="'myitem'"/>

        <t t-call="portal.portal_record_layout">
            <t t-set="card_header">
                <h5 class="mb-0">
                    <t t-out="item.name"/>
                </h5>
            </t>
            <t t-set="card_body">
                <div class="row">
                    <div class="col-12">
                        <!-- Record details here -->
                        <p>Partner: <t t-out="item.partner_id.name"/></p>
                    </div>
                </div>
            </t>
        </t>

        <!-- Chatter — same t-call works v17/v18/v19 -->
        <div id="communication" class="mt-4">
            <t t-call="portal.message_thread">
                <t t-set="object" t-value="item"/>
                <t t-set="token" t-value="token"/>
                <t t-set="pid" t-value="pid"/>
                <t t-set="hash" t-value="hash"/>
                <t t-set="disable_composer" t-value="False"/>
                <t t-set="message_per_page" t-value="10"/>
            </t>
        </div>
    </t>
</template>
```

### Home portal entry (`portal.portal_my_home_menu`)

```xml
<template id="portal_my_home_menu_myitem" name="Portal layout: myitem menu"
          inherit_id="portal.portal_my_home" customize_show="True">
    <xpath expr="//div[hasclass('o_portal_docs')]" position="inside">
        <t t-call="portal.portal_docs_entry">
            <t t-set="title">My Items</t>
            <t t-set="url" t-value="'/my/myitems'"/>
            <t t-set="count" t-value="myitem_count"/>
        </t>
    </xpath>
</template>
```

---

## 16. SignatureForm (v19)

New OWL component for portal signature collection:

```javascript
// portal/static/src/signature_form/signature_form.js
import { Component } from "@odoo/owl";
import { NameAndSignature } from "@web/core/signature/name_and_signature";
import { registry } from "@web/core/registry";

export class SignatureForm extends Component {
    static template = "portal.SignatureForm";
    static components = { NameAndSignature };

    async onSubmit() {
        const { name, signature } = this.nameAndSignatureRef.el;
        await this.rpc('/my/item/sign', {
            res_id: this.props.resId,
            access_token: this.props.accessToken,
            name,
            signature,
        });
        // Redirect or reload
    }
}

registry.category("public_components").add("portal.signature_form", SignatureForm);
```

```xml
<!-- In QWeb template -->
<div t-component="portal.signature_form"
     t-props="{'resId': record.id, 'accessToken': token}"/>
```

---

## 17. Enterprise Portal Features

### Helpdesk (EE — all versions)

```python
# helpdesk/controllers/portal.py
@http.route(['/my/tickets', '/my/tickets/page/<int:page>'],
            type='http', auth="user", website=True)
def my_helpdesk_tickets(self, page=1, ...):
    ...

@http.route(['/helpdesk/ticket/<int:ticket_id>',
             '/helpdesk/ticket/<int:ticket_id>/<access_token>',
             '/my/ticket/<int:ticket_id>'],
            type='http', auth="public", website=True)
def tickets_followup(self, ticket_id=None, access_token=None, **kw):
    ticket_sudo = self._document_check_access('helpdesk.ticket', ticket_id, access_token)
    ...

# Allow portal ticket closing if team setting enabled
@http.route(['/my/ticket/close/<int:ticket_id>/<access_token>'],
            type='http', auth="public", website=True)
def ticket_close(self, ticket_id=None, access_token=None, **kw):
    if ticket_sudo.team_id.allow_portal_ticket_closing:
        ticket_sudo.write({'stage_id': closing_stage.id, 'closed_by_partner': True})
```

### Sign (EE — all versions)

```python
# sign/controllers/portal.py
def _prepare_home_portal_values(self, counters):
    values = super()._prepare_home_portal_values(counters)
    if 'to_sign_count' in counters:
        values['to_sign_count'] = ...
    if 'sign_count' in counters:
        values['sign_count'] = ...
    return values
```

### Sale portal (Community — all versions)

```python
# sale/controllers/portal.py
# quotation_count, order_count in _prepare_home_portal_values
# /my/quotes, /my/orders list routes
# /my/orders/<int:order_id> detail with accept/decline/pay actions
```

---

## 18. Manifest — Assets

### v17

```python
'assets': {
    'web.assets_frontend': [
        'portal/static/src/js/portal.js',
        'portal/static/src/xml/portal_chatter.xml',
        'portal/static/src/js/portal_composer.js',
    ],
}
```

### v18 / v19 — adds chatter bundles

```python
'assets': {
    'web.assets_frontend': [
        'portal/static/src/js/portal.js',
        'portal/static/src/chatter/boot/boot_service.js',
        # ... OWL chatter components
    ],
    # NEW in v18:
    'portal.assets_chatter': [
        ('include', 'web._assets_helpers'),
        'portal/static/src/chatter/core/**/*',
        'portal/static/src/chatter/frontend/**/*',
    ],
    'portal.assets_chatter_style': [
        # Portal chatter CSS
    ],
}
```

### v19 — adds interactions bundle

```python
'assets': {
    'web.assets_frontend': [
        # ...existing...
        'portal/static/src/interactions/*.js',  # public.interactions components
        'portal/static/src/signature_form/**/*',
    ],
}
```

---

## 19. Common Mistakes & Gotchas

| Mistake | Fix |
|---|---|
| Using `check_access_rights` + `check_access_rule` in v18+ | Use `check_access('read')` |
| Referencing `MANDATORY_BILLING_FIELDS` class constant in v18+ | Call `self._get_mandatory_fields()` |
| Using `/portal/attachment/add` in v18+ | Use `/mail/attachment/upload` |
| Not passing `object` to `portal.message_thread` | Always set `object = record_sudo` |
| Missing `pid`/`hash` for chatter auth | Required for non-public users to post messages |
| auth="user" on detail pages | Use auth="public" + `_document_check_access` for shareable URLs |
| Not granting portal group in `ir.model.access.csv` | Add `base.group_portal` row with read=1 |
| Not adding model to ir.rule for portal | Portal users see ALL records without a domain rule |
| Missing `_compute_access_url` override | Default returns `#`, portal URL won't work |
| Counter method name not ending in `_count` | v18+ session caching only works for keys ending `_count` |
