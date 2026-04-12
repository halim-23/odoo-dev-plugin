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

| `auth=` | Who can call |
|---------|-------------|
| `'user'` | authenticated user (session or API key) |
| `'public'` | any visitor (env user = public user) |
| `'none'` | no session — use `sudo()` manually |
| `'api_key'` (v17+) | API key header `Authorization: Bearer <key>` |

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

## REST-style JSON endpoint pattern (v17+)

```python
@http.route('/api/v1/my_model', type='json', auth='api_key',
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

**v17:** `auth='api_key'` officially supported via `res.users.apikeys`.
**v17:** `type='json'` routes auto-handle CSRF internally; `csrf=False` only needed for external callers.
**v18:** `request.get_http_params()` helper for mixed GET/POST params.
**v19:** Route caching headers (`@http.route(..., cache=300)`) for public pages.

## Inherit Existing Controller

```python
from odoo.addons.web.controllers.main import Home

class CustomHome(Home):
    @http.route('/web', type='http', auth='user', website=False)
    def index(self, **kwargs):
        # Custom logic before
        response = super().index(**kwargs)
        return response
```
