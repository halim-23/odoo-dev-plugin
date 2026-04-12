---
name: odoo-view
description: >
  Odoo XML view development for versions 17.0, 18.0, 19.0 (Community & Enterprise).
  Full deprecation knowledge: attrs/states removal, tree→list migration, kanban-box→card,
  column_invisible, optional columns, decoration changes, all removed view attributes.
  Covers form, list, kanban, search, pivot, graph, activity, gantt (EE), map (EE),
  view inheritance, ir.actions. Use when user asks about Odoo views, XML, UI, or invokes /odoo-view.
---

Expert Odoo XML view developer. Full deprecation knowledge v16→v17→v18→v19. CE+EE.

---

## Breaking Changes v16 → v17

### `attrs` REMOVED in v17 ⚠️ CRITICAL

`attrs` is completely removed. All conditional `invisible`, `required`, `readonly` must be inline Python expressions.

```xml
<!-- ❌ v16 pattern — BROKEN in v17 -->
<field name="date_end" attrs="{'invisible': [('type', '!=', 'fixed')], 'required': [('type', '=', 'fixed')]}"/>
<button name="action_confirm" attrs="{'invisible': [('state', '!=', 'draft')]}"/>

<!-- ✅ v17+ pattern — inline expressions -->
<field name="date_end"
       invisible="type != 'fixed'"
       required="type == 'fixed'"/>
<button name="action_confirm" invisible="state != 'draft'"/>
```

**Expression syntax rules:**
- Plain Python boolean expression (not domain list)
- Access fields by name directly: `state`, `type`, `amount_total`
- Use `not`, `and`, `or`, `in`, `not in`, `==`, `!=`, `<`, `>`, etc.
- String comparisons: `state in ('done', 'cancel')`
- False-y check: `invisible="not partner_id"` or `invisible="partner_id == False"`

```xml
<!-- Complex conditions -->
<field name="discount" invisible="pricelist_id.discount_policy == 'without_discount'"/>
<field name="tax_ids" required="fiscal_position_id and fiscal_position_id.auto_apply"/>
<button name="action_validate"
        invisible="state != 'confirm' or not line_ids"/>
```

### `states` REMOVED in v17 ⚠️

```xml
<!-- ❌ v16 — BROKEN in v17 -->
<field name="name" states="draft"/>
<button name="btn" states="draft,sent"/>

<!-- ✅ v17+ -->
<field name="name" invisible="state not in ('draft',)"/>
<button name="btn" invisible="state not in ('draft', 'sent')"/>
```

### When is the error thrown?

`ValidationError` is raised at **view load/render time** (`_validate_view` during `read_combined`), NOT at save time.  
A view with `attrs`/`states` can be saved to the database — it only breaks when the form is opened.

Exact error message (same in v17/v18/v19):
```
Since 17.0, the "attrs" and "states" attributes are no longer used.
View: <view name> in <file path>
```

### Expression eval context — available variables

Variables accessible inside `invisible`, `readonly`, `required` expressions (source: `getBasicEvalContext`):

| Variable | v17 | v18 | v19 | Notes |
|----------|-----|-----|-----|-------|
| field values (direct) | ✅ | ✅ | ✅ | `state`, `partner_id`, `amount_total`, etc. |
| `uid` | ✅ | ✅ | ✅ | current user ID (int) |
| `context` | ✅ | ✅ | ✅ | the view context dict |
| `allowed_company_ids` | ✅ | ✅ | ✅ | list of enabled company IDs |
| `current_company_id` | ✅ | ✅ | ✅ | active company ID |
| `active_id` | ⚠️ v17 deprecated | ❌ REMOVED | ❌ REMOVED | use field value directly |
| `active_ids` | ⚠️ v17 deprecated | ❌ REMOVED | ❌ REMOVED | — |
| `active_model` | ⚠️ v17 deprecated | ❌ REMOVED | ❌ REMOVED | — |

Source-verified: v17 `getBasicEvalContext` includes `active_id` with comment "deprecated, will be removed in v18". v18/v19 `getBasicEvalContext` does not include them.

```xml
<!-- ✅ context.get() works in all versions -->
<field name="price" invisible="not context.get('show_price')"/>

<!-- ✅ uid works in all versions -->
<button name="action_admin_only" invisible="uid != 1"/>

<!-- ❌ active_id in invisible — broken in v18+ -->
<field name="name" invisible="active_id == False"/>
<!-- ✅ fix: use the record field directly -->
<field name="name" invisible="not id"/>
```

### `groups` attribute — still works (all versions)

`groups` is server-side access control — **not** a replacement for `invisible`, but different purpose.

```xml
<!-- Hide field entirely from users not in group (server strips the node) -->
<field name="margin" groups="account.group_show_line_subtotals_tax_selection"/>

<!-- Show button only to managers -->
<button name="action_force_close" string="Force Close"
        groups="base.group_system"/>

<!-- groups on page — entire tab removed for non-members -->
<page string="Accounting" groups="account.group_account_user">
    ...
</page>
```

`invisible` vs `groups`:
- `invisible="..."` — field/button rendered in DOM but hidden; client evaluates the expression
- `groups="..."` — node stripped from arch on server; non-members never see the element at all
- Use `groups` for security-sensitive fields; use `invisible` for UX conditional display

### `<tree>` → `<list>` — renamed in v17, fully adopted in v18

**Source-verified**: v17 core Odoo views (27 files) still use `<tree>` tags internally.  
The v17 view parser accepts both and converts between them.  
**Odoo itself fully converted to `<list>` in v18.** All new code must use `<list>`.

```xml
<!-- ❌ Old tag — still renders in v17/v18 via alias, but DO NOT use in new code -->
<tree string="My Records" ...>

<!-- ✅ v17+ new code — required in v18+ -->
<list string="My Records" ...>
```

**`<tree>` inside form (One2many):** use `<list>` there too.

### `colors` and `fonts` attributes on list — REMOVED v17

```xml
<!-- ❌ v16 — REMOVED -->
<tree colors="red:state=='cancel';green:state=='done'" fonts="bold:state=='confirm'">

<!-- ✅ v17+ — use decoration-* -->
<list decoration-danger="state == 'cancel'"
      decoration-success="state == 'done'"
      decoration-bf="state == 'confirm'">
```

**Decoration classes:**

| Attribute | Bootstrap class | Use for |
|-----------|-----------------|---------|
| `decoration-danger` | `text-danger` | errors, cancelled |
| `decoration-warning` | `text-warning` | warnings, pending |
| `decoration-success` | `text-success` | done, active |
| `decoration-info` | `text-info` | informational |
| `decoration-muted` | `text-muted` | archived, inactive |
| `decoration-bf` | `fw-bold` | emphasis |
| `decoration-it` | `fst-italic` | secondary info |

### Kanban `t-name="kanban-box"` → `t-name="card"` — deprecated in v18

**Source-verified** (`ir_ui_view.py` v18 line 1513): v18 logs a warning when `kanban-box` is detected.  
v17 core still uses `kanban-box` (no warning). v18 core views use `t-name="card"`.

```xml
<!-- ❌ v17 style — deprecated in v18 (warning logged), avoid in all new code -->
<templates>
    <t t-name="kanban-box">
        <div class="oe_kanban_global_click">
            <div class="oe_kanban_content">
                <field name="name"/>
            </div>
        </div>
    </t>
</templates>

<!-- ✅ v18+ required (v17: recommended, no warning) -->
        <aside class="o_kanban_aside_full">
            <field name="image" widget="image" class="o_image_64_cover"/>
        </aside>
        <main>
            <strong class="o_kanban_record_title"><field name="name"/></strong>
            <div class="o_kanban_record_subtitle"><field name="partner_id"/></div>
            <div class="o_kanban_record_bottom">
                <div class="oe_kanban_bottom_left">
                    <field name="priority" widget="priority"/>
                </div>
                <div class="oe_kanban_bottom_right">
                    <field name="activity_ids" widget="kanban_activity"/>
                    <field name="state" widget="label_selection"
                           options="{'classes': {'draft': 'default', 'done': 'success'}}"/>
                </div>
            </div>
        </main>
    </t>
</templates>
```

### `column_invisible` attribute (v17+) — replaces `attrs` on `<field>` inside `<list>`

```xml
<!-- ❌ v16 — column visibility in list via attrs -->
<list>
    <field name="user_id" attrs="{'column_invisible': [('parent.type', '=', 'service')]}"/>
</list>

<!-- ✅ v17+ — direct attribute on field inside list -->
<list>
    <field name="user_id" column_invisible="parent.type == 'service'"/>
</list>
```

`column_invisible` hides the entire column. `invisible` only hides individual cells.

### `optional` columns in list (v17+)

```xml
<list>
    <field name="name"/>
    <field name="partner_id"/>
    <field name="date" optional="hide"/>    <!-- hidden by default, user can show -->
    <field name="notes" optional="show"/>   <!-- shown by default, user can hide -->
    <!-- no optional = always visible, cannot be toggled -->
</list>
```

### `editable` on list (v17+ behaviour)

```xml
<list editable="top">    <!-- new row at top -->
<list editable="bottom"> <!-- new row at bottom (default for O2M) -->
```

`editable` on standalone list actions: editing in-place without opening form.

---

## Breaking Changes v17 → v18

### `<list>` fully adopted in v18 — `<tree>` aliased in both v17 and v18

**Source-verified**: v17 core views still use `<tree>` (27 files). v18 core fully converted to `<list>`.  
In both v17 and v18, `<tree>` still renders via internal alias, but **all new code must use `<list>`**.  
Third-party modules using `<tree>` get deprecation warnings in v18 server logs.

### `optional` columns — improved UX (v18)

Column visibility preferences are now saved per user per view (was per-browser in v17).

### `multi_edit` default behavior (v18)

`multi_edit="1"` on `<list>` now enabled by default for fields that support it.  
Explicitly set `multi_edit="0"` to disable.

### `row_class` attribute (v18)

```xml
<!-- Programmatic row class from computed field -->
<list row_class="row_css_class">
    <field name="row_css_class" column_invisible="1"/>
    <field name="name"/>
</list>
```

`row_css_class` is a Char computed field returning Bootstrap class names.

### Form view `invisible` on `<group>` and `<page>` (v18 improved)

```xml
<!-- v18: invisible on notebook page collapses the tab -->
<notebook>
    <page string="Enterprise" invisible="not is_enterprise" name="enterprise_tab">
        ...
    </page>
</notebook>

<!-- v18: invisible on <group> hides the entire group -->
<group invisible="type != 'service'">
    <field name="service_type"/>
    <field name="service_duration"/>
</group>
```

### `web_ribbon` widget (v17+ / v18 improved)

```xml
<!-- Shows colored ribbon (replaces old oe_ribbon pattern) -->
<widget name="web_ribbon" title="Archived" bg_color="text-bg-danger" invisible="active"/>
<widget name="web_ribbon" title="New" bg_color="text-bg-success" invisible="not is_new"/>
```

---

## Breaking Changes v18 → v19

### `row_click_action` on `<list>` (v19)

```xml
<!-- Override what happens when clicking a row -->
<list row_click_action="action_open_detail">
    <field name="name"/>
</list>
```

`action_open_detail` is a Python method returning an action dict.

### `<form>` `auto_focus` attribute (v19)

```xml
<!-- Focus first visible required field automatically -->
<form auto_focus="1">
```

---

## Form View (v17+ complete)

```xml
<record id="view_my_model_form" model="ir.ui.view">
    <field name="name">my.model.form</field>
    <field name="model">my.model</field>
    <field name="arch" type="xml">
        <form>
            <header>
                <button name="action_confirm" string="Confirm" type="object"
                        class="oe_highlight"
                        invisible="state != 'draft'"/>
                <button name="action_reset" string="Reset to Draft" type="object"
                        invisible="state not in ('done', 'cancel')"/>
                <button name="action_cancel" string="Cancel" type="object"
                        invisible="state in ('cancel', 'draft')"
                        confirm="Are you sure you want to cancel?"/>
                <field name="state" widget="statusbar"
                       statusbar_visible="draft,confirm,done"
                       clickable="1"/>
            </header>
            <sheet>
                <!-- Stat buttons -->
                <div class="oe_button_box" name="button_box">
                    <button class="oe_stat_button" type="object"
                            name="action_view_moves" icon="fa-book"
                            invisible="move_count == 0">
                        <field name="move_count" widget="statinfo" string="Moves"/>
                    </button>
                </div>
                <!-- Ribbon for special states -->
                <widget name="web_ribbon" title="Archived"
                        bg_color="text-bg-danger" invisible="active"/>
                <!-- Avatar image -->
                <field name="image" widget="image" class="oe_avatar"
                       options="{'preview_image': 'image_128'}"/>
                <!-- Title -->
                <div class="oe_title">
                    <label for="name"/>
                    <h1>
                        <field name="name" placeholder="Name..."/>
                    </h1>
                </div>
                <!-- Main fields -->
                <group>
                    <group name="left">
                        <field name="partner_id"/>
                        <field name="date"/>
                        <field name="company_id" invisible="not is_multi_company"/>
                    </group>
                    <group name="right">
                        <field name="amount_total" widget="monetary"/>
                        <field name="currency_id" invisible="1"/>
                        <field name="state" invisible="1"/>
                    </group>
                </group>
                <!-- Lines -->
                <notebook>
                    <page string="Order Lines" name="lines">
                        <field name="line_ids" widget="section_and_note_one2many">
                            <list editable="bottom">
                                <field name="sequence" widget="handle"/>
                                <field name="product_id"/>
                                <field name="qty"/>
                                <field name="price_unit"/>
                                <field name="tax_ids" widget="many2many_tags"/>
                                <field name="subtotal" column_invisible="parent.show_subtotal == False"/>
                            </list>
                        </field>
                    </page>
                    <page string="Notes" name="notes">
                        <field name="notes" widget="html" placeholder="Internal notes..."/>
                    </page>
                    <page string="Other Info" name="other">
                        <group>
                            <field name="user_id"/>
                            <field name="ref"/>
                        </group>
                    </page>
                </notebook>
            </sheet>
            <!-- Chatter: v17 pattern — use <chatter/> for v18/v19 (see Chatter section below) -->
            <div class="oe_chatter">
                <field name="message_follower_ids"/>
                <field name="activity_ids"/>
                <field name="message_ids"/>
            </div>
        </form>
    </field>
</record>
```

---

## Chatter in Form Views — Version Changes (source-verified)

### v17 — `<div class="oe_chatter">` with 3 required child fields

```xml
<form>
    <sheet>
        ...
    </sheet>
    <div class="oe_chatter">
        <field name="message_follower_ids"/>
        <field name="activity_ids"/>
        <field name="message_ids"/>
    </div>
</form>
```

Form compiler selector: `div.oe_chatter` — reads child `<field>` nodes to derive Chatter OWL props:
- `message_follower_ids` → `hasFollowers=true`
- `activity_ids` → `hasActivities=true`
- `message_ids` → `hasMessageList=true`

Options in v17 set on the field nodes:
```xml
<field name="message_follower_ids" options="{'post_refresh': True}"/>
<field name="message_ids" options="{'post_refresh': 'always', 'open_attachments': True}"/>
```

### v18 / v19 — `<chatter/>` single self-closing tag (BREAKING CHANGE)

```xml
<form>
    <sheet>
        ...
    </sheet>
    <chatter/>
</form>
```

- Form compiler selector changed to `"chatter"` — `div.oe_chatter` compiler is **NOT registered** in v18/v19
- **`<div class="oe_chatter">` is silently ignored in v18/v19** — chatter won't appear
- All 3 child fields gone — `has_activities` is auto-detected from `archInfo` (set if model inherits `mail.activity.mixin`)
- Options moved to tag attributes (see table below)

### `<chatter>` attributes — source-verified from `form_compiler.js`

| Attribute | v18 | v19 | Effect |
|-----------|-----|-----|--------|
| `reload_on_post="True"` | ✅ | ✅ | Reload form when message posted |
| `reload_on_attachment="True"` | ✅ | ✅ | Reload when attachment changes |
| `reload_on_follower="True"` | ✅ | ✅ | Reload when follower list changes |
| `open_attachments="True"` | ✅ | ✅ | Show attachment box by default |
| `reload_on_activity="True"` | ❌ | ✅ v19 NEW | Reload when activity changes |

```xml
<!-- Standard -->
<chatter/>

<!-- Open attachment box on load -->
<chatter open_attachments="True"/>

<!-- Portal/public forms: reload after post -->
<chatter reload_on_post="True"/>

<!-- v19: reload on activity change too -->
<chatter reload_on_post="True" reload_on_activity="True"/>
```

Source: `event/views/event_track_views.xml` uses `reload_on_post="True"` in both v18 and v19.

### Position rule — chatter must be OUTSIDE `<sheet>` (all versions)

```xml
<!-- ✅ Correct — all versions -->
<form>
    <sheet>...</sheet>
    <chatter/>
</form>

<!-- ❌ Wrong — chatter inside sheet breaks aside/bottom layout logic -->
<form>
    <sheet>
        <chatter/>
    </sheet>
</form>
```

Source: `form_compiler.js`: `if (parentNode.classList.contains("o_form_sheet")) { return res; }` — compiler skips layout repositioning when chatter is inside sheet.

### Inheriting a view to add chatter

```xml
<!-- v17: xpath after sheet, append oe_chatter div -->
<xpath expr="//sheet" position="after">
    <div class="oe_chatter">
        <field name="message_follower_ids"/>
        <field name="activity_ids"/>
        <field name="message_ids"/>
    </div>
</xpath>

<!-- v18/v19: xpath after sheet, append chatter tag -->
<xpath expr="//sheet" position="after">
    <chatter/>
</xpath>
```

### Summary

| | v17 | v18 | v19 |
|-|-----|-----|-----|
| Syntax | `<div class="oe_chatter">` | `<chatter/>` | `<chatter/>` |
| Child fields needed | 3 required | none | none |
| `oe_chatter` div works | ✅ | ❌ silently ignored | ❌ silently ignored |
| Options via | field `options={}` | tag attributes | tag attributes |
| `has_activities` source | child field presence | `archInfo` auto | `archInfo` auto |
| `reload_on_activity` attr | ❌ | ❌ | ✅ |

---

## List View (v17+)

```xml
<record id="view_my_model_list" model="ir.ui.view">
    <field name="name">my.model.list</field>
    <field name="model">my.model</field>
    <field name="arch" type="xml">
        <list string="My Models"
              decoration-danger="state == 'cancel'"
              decoration-success="state == 'done'"
              decoration-muted="not active"
              multi_edit="1"
              expand="1">
            <field name="name"/>
            <field name="partner_id"/>
            <field name="date" optional="show"/>
            <field name="user_id" optional="hide"/>
            <field name="amount_total" sum="Total" avg="Average"/>
            <field name="state" widget="badge"
                   decoration-success="state == 'done'"
                   decoration-warning="state == 'confirm'"
                   decoration-danger="state == 'cancel'"/>
            <field name="active" column_invisible="1"/>
        </list>
    </field>
</record>
```

---

## Kanban View (v17+)

```xml
<record id="view_my_model_kanban" model="ir.ui.view">
    <field name="name">my.model.kanban</field>
    <field name="model">my.model</field>
    <field name="arch" type="xml">
        <kanban default_group_by="state" quick_create="false"
                records_draggable="1" group_create="false">
            <field name="name"/>
            <field name="partner_id"/>
            <field name="state"/>
            <field name="amount_total"/>
            <field name="priority"/>
            <field name="activity_ids"/>
            <field name="color"/>
            <templates>
                <!-- v17+: t-name="card" replaces t-name="kanban-box" -->
                <t t-name="card"
                   t-att-class="record.color.raw_value ? 'oe_kanban_color_' + record.color.raw_value : ''">
                    <main>
                        <strong class="o_kanban_record_title">
                            <field name="name"/>
                        </strong>
                        <div class="o_kanban_record_subtitle text-muted">
                            <field name="partner_id"/>
                        </div>
                        <div class="o_kanban_record_bottom mt-2">
                            <div class="oe_kanban_bottom_left">
                                <field name="priority" widget="priority"/>
                                <field name="amount_total" widget="monetary"/>
                            </div>
                            <div class="oe_kanban_bottom_right">
                                <field name="activity_ids" widget="kanban_activity"/>
                                <field name="state" invisible="1"/>
                            </div>
                        </div>
                    </main>
                </t>
            </templates>
        </kanban>
    </field>
</record>
```

---

## Search View (v17+)

```xml
<record id="view_my_model_search" model="ir.ui.view">
    <field name="name">my.model.search</field>
    <field name="model">my.model</field>
    <field name="arch" type="xml">
        <search string="Search My Models">
            <!-- Multi-field search -->
            <field name="name" string="Name / Ref"
                   filter_domain="['|', ('name', 'ilike', self), ('ref', 'ilike', self)]"/>
            <field name="partner_id"/>
            <separator/>
            <!-- Filters -->
            <filter name="draft" string="Draft" domain="[('state', '=', 'draft')]"/>
            <filter name="confirmed" string="Confirmed" domain="[('state', '=', 'confirm')]"/>
            <filter name="done" string="Done" domain="[('state', '=', 'done')]"/>
            <separator/>
            <filter name="my_records" string="My Records"
                    domain="[('user_id', '=', uid)]"/>
            <filter name="today" string="Today"
                    domain="[('date', '=', context_today().strftime('%Y-%m-%d'))]"/>
            <filter name="this_week" string="This Week"
                    domain="[('date', '&gt;=', (context_today() - datetime.timedelta(days=7)).strftime('%Y-%m-%d'))]"/>
            <separator/>
            <filter name="inactive" string="Archived" domain="[('active', '=', False)]"/>
            <!-- Group By -->
            <group expand="0" string="Group By">
                <filter name="by_partner" string="Partner"
                        context="{'group_by': 'partner_id'}"/>
                <filter name="by_state" string="Status"
                        context="{'group_by': 'state'}"/>
                <filter name="by_date" string="Date"
                        context="{'group_by': 'date:month'}"/>
                <filter name="by_week" string="Week"
                        context="{'group_by': 'date:week'}"/>
            </group>
            <!-- Default filter -->
            <searchpanel>
                <field name="state" select="multi" icon="fa-circle" enable_counters="1"/>
                <field name="partner_id" select="one" icon="fa-user"/>
            </searchpanel>
        </search>
    </field>
</record>
```

---

## View Inheritance (v17+)

```xml
<record id="view_my_model_form_inherit_mymodule" model="ir.ui.view">
    <field name="name">my.model.form.inherit.mymodule</field>
    <field name="model">my.model</field>
    <field name="inherit_id" ref="sale.view_order_form"/>
    <field name="arch" type="xml">

        <!-- Add field AFTER existing field -->
        <field name="partner_id" position="after">
            <field name="my_custom_field"/>
        </field>

        <!-- Add field BEFORE -->
        <field name="date_order" position="before">
            <field name="my_date_field"/>
        </field>

        <!-- Add to button_box -->
        <div name="button_box" position="inside">
            <button class="oe_stat_button" type="object"
                    name="action_my_view" icon="fa-star">
                <field name="my_count" widget="statinfo" string="Items"/>
            </button>
        </div>

        <!-- Change attribute -->
        <field name="name" position="attributes">
            <attribute name="required">1</attribute>
            <attribute name="placeholder">Custom placeholder</attribute>
        </field>

        <!-- Replace element entirely -->
        <field name="old_field" position="replace">
            <field name="new_field"/>
        </field>

        <!-- Remove element -->
        <field name="unwanted_field" position="replace"/>

        <!-- Add inside notebook -->
        <notebook position="inside">
            <page string="My Custom Tab" name="my_tab">
                <field name="my_tab_field"/>
            </page>
        </notebook>

        <!-- Add after a specific page -->
        <page name="other_info" position="after">
            <page string="My Info" name="my_info">
                <field name="info_field"/>
            </page>
        </page>

        <!-- XPath for complex targeting -->
        <xpath expr="//field[@name='order_line']/list/field[@name='product_id']" position="after">
            <field name="custom_line_field"/>
        </xpath>

        <!-- XPath to add class to existing element -->
        <xpath expr="//div[@class='oe_title']" position="attributes">
            <attribute name="class" add="my_custom_class" separator=" "/>
        </xpath>

    </field>
</record>
```

---

## Actions & Menus

```xml
<!-- Window action -->
<record id="action_my_model" model="ir.actions.act_window">
    <field name="name">My Models</field>
    <field name="res_model">my.model</field>
    <field name="view_mode">list,form,kanban</field>
    <field name="context">{'default_state': 'draft', 'search_default_my_records': 1}</field>
    <field name="domain">[('active', '=', True)]</field>
    <field name="help" type="html">
        <p class="o_view_nocontent_smiling_face">Create your first record</p>
        <p>Your records will appear here once created.</p>
    </field>
</record>

<!-- Server action (call method from action menu) -->
<record id="action_server_confirm" model="ir.actions.server">
    <field name="name">Confirm</field>
    <field name="model_id" ref="model_my_model"/>
    <field name="binding_model_id" ref="model_my_model"/>
    <field name="state">code</field>
    <field name="code">records.action_confirm()</field>
</record>

<!-- Menu items -->
<menuitem id="menu_root" name="My App" sequence="100" web_icon="my_module,static/description/icon.png"/>
<menuitem id="menu_my_model" name="My Models" parent="menu_root" action="action_my_model" sequence="10"/>
<menuitem id="menu_config" name="Configuration" parent="menu_root" sequence="99"
          groups="my_module.group_my_module_manager"/>
```

---

## Widget Reference (v17+)

### Standard Widgets

| Widget | Field type | Notes |
|--------|-----------|-------|
| `monetary` | Float/Monetary | needs `currency_field` |
| `statusbar` | Selection | header status strip |
| `badge` | Selection | colored pill |
| `many2many_tags` | Many2many | inline tags |
| `many2many_checkboxes` | Many2many | checkbox list |
| `radio` | Selection | radio buttons |
| `priority` | Selection (0-3) | star rating |
| `handle` | Integer | drag-to-sort |
| `html` | Html | rich text editor (Odoo editor) |
| `image` | Binary/Image | image preview |
| `binary` | Binary | file upload/download |
| `url` | Char | clickable link |
| `email` | Char | mailto link |
| `phone` | Char | tel link |
| `float_time` | Float | display as HH:MM |
| `percentpie` | Float | pie chart 0-100 |
| `gauge` | Float | gauge chart |
| `progressbar` | Float | progress bar |
| `color_picker` | Integer | color picker (0-11) |
| `ace` | Text/Html | code editor |
| `domain` | Char | domain editor UI |
| `reference` | Reference | model + record picker |
| `statinfo` | Integer/Float | stat button counter |
| `kanban_activity` | Many2many | activity dot in kanban |
| `label_selection` | Selection | colored label |
| `boolean_toggle` | Boolean | toggle switch |
| `boolean_favorite` | Boolean | star icon |
| `section_and_note_one2many` | One2many | lines with section/note rows |
| `signature` | Html/Binary | signature capture (EE) |

### Deprecated/Removed Widgets (v17+)

| Widget | Status | Replacement |
|--------|--------|-------------|
| `many2one_barcode` | removed v17 | custom OWL widget |
| `pad` | removed v17 (EE pad removed) | `html` widget |
| `color` | renamed | `color_picker` |
| `radio_image` | removed v17 | `radio` |
| `hr_org_chart` | moved to HR module | still available if hr installed |

---

## Full Removed Attributes Table

| Attribute | Removed | Notes |
|-----------|---------|-------|
| `attrs` | v17 | use `invisible`, `required`, `readonly` inline |
| `states` | v17 | use `invisible="state not in (...)"` |
| `colors` on `<list>` | v17 | use `decoration-*` |
| `fonts` on `<list>` | v17 | use `decoration-bf`, `decoration-it` |
| `t-name="kanban-box"` | v18 | use `t-name="card"` (warning logged v18+) |
| `oe_kanban_global_click` css class | v17 | cards are clickable by default |
| `on_change` attribute | v10 | use `@api.onchange` in Python |
| `select` attribute on field | v9 | search view filter |
| `nolabel` replaced by | — | `<label for="" string=""/>` with empty string |
| `string=""` on group | — | hides group label border |
| `widget="selection"` | v17 | use `widget="radio"` or plain Selection |
