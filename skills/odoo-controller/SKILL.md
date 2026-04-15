---
name: odoo-controller
description: >
  Odoo HTTP and JSON-RPC controller development for versions 17.0, 18.0, 19.0 (Community & Enterprise).
  Covers route definitions, authentication types, JSON endpoints, website pages, portal pages,
  file downloads, and CORS. Use when user asks about Odoo controllers, routes, HTTP, REST API,
  or invokes /odoo-controller.
---

Expert Odoo controller developer. Target: 17/18/19 CE+EE.

## Basic Controller

```python
from odoo import http
from odoo.http import request, Response
import json


class MyController(http.Controller):

    # Public JSON endpoint
    @http.route('/api/my_model', type='json', auth='none', methods=['POST'], csrf=False)
    def get_my_model(self, **kwargs):
        records = request.env['my.model'].sudo().search([], limit=10)
        return {'records': records.read(['name', 'state'])}

    # Authenticated JSON endpoint
    @http.route('/api/my_model/create', type='json', auth='user', methods=['POST'])
    def create_my_model(self, name, **kwargs):
        rec = request.env['my.model'].create({'name': name})
        return {'id': rec.id, 'name': rec.name}

    # HTML page (public website)
    @http.route('/my-page', type='http', auth='public', website=True)
    def my_page(self, **kwargs):
        records = request.env['my.model'].sudo().search([('state', '=', 'done')])
        return request.render('my_module.my_page_template', {'records': records})

    # Portal page (logged-in user)
    @http.route('/my/portal', type='http', auth='user', website=True)
    def portal_page(self, **kwargs):
        records = request.env['my.model'].search([
            ('partner_id', '=', request.env.user.partner_id.id)
        ])
        return request.render('my_module.portal_page_template', {'records': records})

    # File download
    @http.route('/my_model/<int:record_id>/download', type='http', auth='user')
    def download_file(self, record_id, **kwargs):
        record = request.env['my.model'].browse(record_id)
        if not record.exists():
            return request.not_found()
        file_data = record.attachment_id.raw
        return Response(
            file_data,
            headers=[
                ('Content-Type', 'application/pdf'),
                ('Content-Disposition', f'attachment; filename="{record.name}.pdf"'),
            ]
        )
```

## Auth Types

| `auth=` | Who can call | Version |
|---------|-------------|---------|
| `'user'` | authenticated user (session) | all |
| `'public'` | any visitor (env user = public user) | all |
| `'none'` | no session — use `sudo()` manually | all |
| `'api_key'` | API key via `Authorization: Bearer <key>` (legacy alias) | v17 |
| `'bearer'` | Bearer token auth; falls back to session if no header | v18+ |

> **v17 vs v18+ auth:** In v17, `auth='api_key'` is the token auth mechanism. In v18+, it is replaced/superseded by `auth='bearer'`, which also falls back to a normal session if no `Authorization` header is present. Source: `odoo/http.py` docstring for `route()` in each version.

## JSON-RPC (Odoo native RPC)

Odoo uses its own JSON-RPC 2.0 envelope at `/web/dataset/call_kw`:

```javascript
// From browser / external client
fetch('/web/dataset/call_kw', {
  method: 'POST',
  headers: {'Content-Type': 'application/json'},
  body: JSON.stringify({
    jsonrpc: '2.0', method: 'call', id: 1,
    params: {
      model: 'my.model',
      method: 'search_read',
      args: [[['state', '=', 'draft']]],
      kwargs: {fields: ['name', 'state'], limit: 5},
    }
  })
})
```

## REST-style JSON endpoint pattern

```python
# v17 — auth='api_key'
@http.route('/api/v1/my_model', type='json', auth='api_key',
            methods=['GET', 'POST'], csrf=False, cors='*')

# v18+ — auth='bearer' (falls back to session if no Authorization header)
@http.route('/api/v1/my_model', type='json', auth='bearer',
            methods=['GET', 'POST'], csrf=False, cors='*')
def api_my_model(self, **kwargs):
    method = request.httprequest.method
    if method == 'GET':
        domain = kwargs.get('domain', [])
        recs = request.env['my.model'].search_read(domain, ['name', 'state'])
        return recs
    elif method == 'POST':
        rec = request.env['my.model'].create(kwargs.get('values', {}))
        return {'id': rec.id}
```

## Error Handling in JSON

```python
from odoo.exceptions import UserError, AccessError

@http.route('/api/safe', type='json', auth='user')
def safe_endpoint(self, **kwargs):
    try:
        result = request.env['my.model'].do_something()
        return {'success': True, 'data': result}
    except AccessError as e:
        return {'success': False, 'error': 'access_denied', 'message': str(e)}
    except UserError as e:
        return {'success': False, 'error': 'user_error', 'message': str(e)}
```

## Website QWeb Template

```xml
<template id="my_page_template" name="My Page">
    <t t-call="website.layout">
        <div id="wrap">
            <div class="container">
                <h1>My Records</h1>
                <t t-foreach="records" t-as="rec">
                    <div class="card mb-2">
                        <div class="card-body">
                            <h5 t-field="rec.name"/>
                        </div>
                    </div>
                </t>
            </div>
        </div>
    </t>
</template>
```

## Version Notes

**v17:** `auth='api_key'` is the token auth mechanism.
**v17:** `type='json'` routes auto-handle CSRF internally; `csrf=False` only needed for external callers.
**v18:** `request.get_http_params()` helper for merged GET query string + form body params.
**v18:** `readonly=True` route parameter — opens a cursor on the read-only replica instead of primary DB.
**v19:** `type='json'` is a **deprecated alias** for `type='jsonrpc'` — use `type='jsonrpc'` in new code.
Source: `odoo/http.py` `route()` with `DeprecationWarning` in v19.
**v19:** New `captcha=` route param for captcha validation; `save_session=` controls session cookie
(defaults to `False` for `auth='bearer'`).
Source: `odoo/http.py` `route()` docstring in v19.

## v17 vs v18+ URL Routing Differences

### 1. `reroute()` moved from `IrHttp` classmethod → `request` method

```python
# v17 — classmethod on ir.http (http_routing addon)
IrHttp.reroute('/path/without/lang')

# v18+ — method on the request object (odoo/http.py)
request.reroute('/path/without/lang')
```

Source: `addons/http_routing/models/ir_http.py` (v17 `cls.reroute()`),
`odoo/http.py:1976` (v18 `Request.reroute()`).

### 2. `slug` / `unslug` / `url_for` / `url_lang` moved from module functions → `ir.http` classmethods

```python
# v17 — module-level functions
from odoo.addons.http_routing.models.ir_http import (
    slug, slugify, unslug, unslug_url,
    url_for, url_lang, is_multilang_url,
)
name = slug(record)
path = url_for('/my-page')
multilang = is_multilang_url('/my-page')

# v18+ — classmethods on ir.http (use via env)
IrHttp = request.env['ir.http']
name    = IrHttp._slug(record)          # or (id, name) tuple
path    = IrHttp._url_for('/my-page')
lang_p  = IrHttp._url_lang('/my-page')
multilang = IrHttp._is_multilang_url('/my-page')
text    = IrHttp._slugify('Some Name')
id, slug_str = IrHttp._unslug('my-record-42')
```

Source: `addons/http_routing/models/ir_http.py` — v17 has top-level `slug()`, `url_for()`, etc.;
v18+ moves them under `class IrHttp` as `_slug`, `_url_for`, `_url_lang`, `_is_multilang_url`,
`_slugify`, `_unslug`, `_unslug_url` (prefixed with `_` to mark as private API).

### 3. `request.lang` type changed: ORM record → `LangData` namedtuple

```python
# v17 — request.lang is a res.lang ORM record
lang_code     = request.lang._get_cached('code')      # e.g. 'fr_BE'
lang_url_code = request.lang._get_cached('url_code')  # e.g. 'fr'

# v18+ — request.lang is a LangData (ReadonlyDict / namedtuple-like)
lang_code     = request.lang.code      # direct attribute access
lang_url_code = request.lang.url_code
```

Source: v17 `addons/http_routing/models/ir_http.py` uses `_get_cached()`;
v18+ uses `LangData` from `odoo/addons/base/models/res_lang.py` with plain attribute access.

### 4. Cookie access shortcut on `request`

```python
# v17 — always go through httprequest
lang_cookie = request.httprequest.cookies.get('frontend_lang')

# v18+ — direct proxy on request
lang_cookie = request.cookies.get('frontend_lang')
```

Source: v18 `addons/http_routing/models/ir_http.py` line ~416 uses `request.cookies`.

### 5. `readonly` route parameter (v18+ only)

Routes that only read data can declare `readonly=True` to open a cursor on the
read-only replica instead of the primary database, reducing load:

```python
# v18+ only
@http.route('/api/products', type='json', auth='public', readonly=True)
def list_products(self, **kwargs):
    return request.env['product.template'].sudo().search_read([], ['name', 'price'])
```

`auth='none'` routes default to `readonly=True`. Source: `odoo/http.py` `route()` docstring
and `_check_and_complete_route_definition()` in v18/v19.

### 6. `res.lang` lookup API changed

```python
# v17
code = request.env['res.lang']._lang_get_code(url_lang_str)
rec  = request.env['res.lang']._lang_get(lang_code)

# v18+
data = request.env['res.lang']._get_data(url_code=url_lang_str)
code = data.code   # LangData attribute
```

Source: `addons/http_routing/models/ir_http.py` `_match()` method in each version.

### 7. Frontend slug regex: ASCII-only → Unicode (v18+)

The `<model(...)>` URL converter uses a slug regex that changed in v18:

```python
# v17 — middle chars ASCII only: [A-Za-z0-9-_]
# Matches: my-product-42, some_thing-7
# Fails:   résumé-10, некий-товар-7 (non-ASCII in middle)

# v18/v19 — middle chars Unicode \w + hyphen: [\w-]
# Matches: résumé-10, café-99, некий-товар-7  ← NEW
```

Source: `addons/http_routing/models/ir_http.py` `_UNSLUG_RE` / `_UNSLUG_ROUTE_PATTERN`
— v17 uses `[A-Za-z0-9-_]`, v18/v19 uses `[\w-]`.

## Inherit Existing Controller

```python
# v17 — Home was re-exported from main.py
from odoo.addons.web.controllers.main import Home

# v18/v19 — import directly from home.py (main.py no longer re-exports Home)
from odoo.addons.web.controllers.home import Home

class CustomHome(Home):
    @http.route('/web', type='http', auth='user', website=False)
    def index(self, **kwargs):
        response = super().index(**kwargs)
        return response
```

## Backend URL Patterns (v17 vs v18+)

### v17 — Hash-based (`/web#key=value`)

All backend state lives in the **URL hash fragment** (never sent to server):

```
/web
/web#action=263
/web#action=sale.action_quotations_with_onboarding
/web#action=263&view_type=form&id=42
/web#model=res.partner&view_type=list
```

### v18/v19 — Path-based (`/odoo/...`)

State moves into the **URL pathname**. `/web` still redirects to `/odoo` for backward compat.

```
/odoo                              # home / dashboard
/odoo/contacts                     # action with path="contacts"
/odoo/contacts/42                  # single record
/odoo/contacts/new                 # new record form
/odoo/42/contacts                  # nested: parent resId / child action
/odoo/action-263                   # action by numeric ID (no path field)
/odoo/action-sale.action_orders    # action by xml_id  (no path field)
/odoo/res.partner                  # model with dot    (no action)
/odoo/m-mymodel                    # model without dot (no action)
/scoped_app/...                    # same paths, PWA standalone mode
```

**URL segment rules** (`router.js` `pathFromActionState()`):

| Segment | Meaning |
|---------|---------|
| `contacts` | `ir.actions.path = 'contacts'` (clean path) |
| `action-263` | action by numeric id |
| `action-sale.action_orders` | action by xml_id |
| `res.partner` | model with a dot |
| `m-mymodel` | model without a dot |
| `42` | record id (`resId`) |
| `new` | new record form |

**Backward compat:** `/web#action=263&id=42` is auto-redirected — `urlToState()` detects
`/web` pathname, remaps `id→resId` and `view_type=form→resId=new`, then rewrites to `/odoo/...`.

**`ir.actions.path` field (v18+ only)** — gives an action a clean URL segment:

```xml
<record id="action_contacts" model="ir.actions.act_window">
    <field name="path">contacts</field>   <!-- /odoo/contacts -->
</record>
```

Pattern constraint: `[a-z][a-z0-9_-]*`, unique across all action types.
Reserved: cannot start with `m-` or `action-`, cannot be `new`.
Source: `odoo/addons/base/models/ir_actions.py` `_check_path()` in v18/v19.
