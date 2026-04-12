---
name: odoo-js
description: >
  Odoo JavaScript and OWL component development for versions 17.0, 18.0, 19.0 (Community & Enterprise).
  Full deprecation knowledge: OWL 1→2 migration, complete legacy widget removal list,
  t.widget→Component, action manager API changes, mount/render API, deprecated hooks,
  service API changes, field widget system (standardFieldProps, useInputField, useRecordObserver,
  extractProps, supportedOptions, formatters/parsers registries), view_widgets registry,
  custom view type registration (ArchParser/Controller/Model/Renderer), doAction complete options,
  reactive/markRaw/toRaw (v18+), t-portal (v19), t-inherit template extension, slot system,
  useSubEnv/useChildSubEnv, useBus, useExternalListener, systray, record API in field widgets,
  archParseBoolean→exprToBoolean change (v18), registry categories reference.
  Use when user asks about Odoo JS, OWL, frontend, widgets, custom fields, views, or invokes /odoo-js.
---

Expert Odoo OWL/JS developer. Full OWL 1→2 deprecation knowledge. v17/18/19 CE+EE.

---

## Breaking Changes: Legacy JS → OWL (v15→v16→v17)

Odoo deprecated the legacy Backbone-based widget system starting v15/v16.  
In v17 it is **strongly discouraged** — a `/legacy/` folder still exists in `web/static/src/` with a compat shim,  
but `odoo.define()` is assigned as a backward-compat wrapper in `module_loader.js`.  
**Do NOT use any legacy APIs in new code.** All new code must be OWL 2 components.

### Legacy Widget System — DEPRECATED v17 (legacy/ folder exists; avoid all new code)

```javascript
// ❌ DEPRECATED — legacy Widget class (legacy/ compat folder exists but avoid)
const MyWidget = Widget.extend({
    template: 'MyWidget',
    events: { 'click .btn': '_onClick' },
    init(parent, options) { this._super(parent); },
    start() { this._super(); },
    _onClick() { }
});

// ❌ DEPRECATED — legacy field widget (avoid; OWL replacement required)
const MyFieldWidget = AbstractField.extend({
    supportedFieldTypes: ['char'],
    _renderEdit() { },
    _renderReadonly() { }
});
field_registry.add('my_widget', MyFieldWidget);

// ❌ DEPRECATED — legacy component mounting (no longer works outside legacy context)
const widget = new MyWidget(this, {});
widget.appendTo(this.$el);

// ❌ DEPRECATED — odoo.define() compat shim exists in module_loader.js but avoid in new code
odoo.define('my_module.MyWidget', function (require) {
    const Widget = require('web.Widget');
    // ...
    return MyWidget;
});
```

### New Module System — `/** @odoo-module **/` (v15+, REQUIRED v17+)

```javascript
// ✅ v17+ — ES module syntax with @odoo-module directive
/** @odoo-module **/

import { Component, useState } from "@odoo/owl";
import { registry } from "@web/core/registry";
```

**Import path aliases (v17+):**

| Alias | Resolves to |
|-------|------------|
| `@odoo/owl` | OWL 2 library |
| `@web/...` | `addons/web/static/src/...` |
| `@web/core/registry` | central service/component registry |
| `@web/core/utils/hooks` | `useService`, `useEnv`, etc. |
| `@web/views/fields/standard_field_props` | field props schema |
| `@web/views/form/form_controller` | FormController |
| `@web/views/list/list_controller` | ListController |
| `@web/core/orm_service` | ORM service |

---

## OWL 1 → OWL 2 Breaking Changes (v15→v16 transition, enforced v17)

### Component mounting

```javascript
// ❌ OWL 1
import { mount } from "@odoo/owl";
mount(App, { target: document.body, props: {} });

// ✅ OWL 2
import { App, whenReady } from "@odoo/owl";
whenReady(() => {
    const app = new App(MyComponent, { props: {}, env });
    app.mount(document.body);
});
```

### `willStart` → `onWillStart`

```javascript
// ❌ OWL 1 lifecycle hooks (class methods)
class MyComp extends Component {
    async willStart() { await this.loadData(); }
    mounted() { this.initChart(); }
    willUnmount() { this.chart.destroy(); }
    patched() { this.updateChart(); }
}

// ✅ OWL 2 — hooks in setup()
class MyComp extends Component {
    setup() {
        onWillStart(async () => { await this.loadData(); });
        onMounted(() => { this.initChart(); });
        onWillUnmount(() => { this.chart.destroy(); });
        onPatched(() => { this.updateChart(); });
    }
}
```

### `useState` / reactivity changes

```javascript
// ❌ OWL 1 — state was a method
class MyComp extends Component {
    constructor() {
        this.state = useState({ count: 0 }); // inside constructor
    }
}

// ✅ OWL 2 — always inside setup()
class MyComp extends Component {
    setup() {
        this.state = useState({ count: 0, records: [] });
    }
}
```

### Props validation

```javascript
// ✅ OWL 2 — props schema (required)
class MyComp extends Component {
    static props = {
        recordId: { type: Number },
        title: { type: String, optional: true },
        onSave: { type: Function, optional: true },
        items: { type: Array, element: Object },
        config: {
            type: Object,
            shape: {
                mode: { type: String, optional: true },
                limit: Number,
            },
            optional: true,
        },
    };
    static defaultProps = { title: "Default" };
}
```

### Event handling — `t-on-event` (kebab-case)

```xml
<!-- ❌ OWL 1 (PascalCase event names) -->
<button t-on-Click="onClick">Click</button>

<!-- ✅ OWL 2 (kebab-case) -->
<button t-on-click="onClick">Click</button>
<input t-on-input="onInput" t-on-keydown="onKeydown"/>
<div t-on-mouseenter="onHover" t-on-mouseleave="onLeave"/>
```

### `t-component` dynamic components

```xml
<!-- ✅ OWL 2 dynamic component -->
<t t-component="dynamicComponent" t-props="componentProps"/>
```

### `useRef` — accessing DOM

```javascript
// ✅ OWL 2
setup() {
    this.inputRef = useRef("myInput");
    onMounted(() => {
        this.inputRef.el?.focus();
    });
}
```
```xml
<input t-ref="myInput" type="text"/>
```

### `t-key` required for lists

```xml
<!-- ✅ Always provide t-key when iterating -->
<t t-foreach="items" t-as="item" t-key="item.id">
    <div t-esc="item.name"/>
</t>
```

---

## OWL 2 Hooks Reference (v17+)

```javascript
import {
    Component,
    useState,          // reactive proxy object — mutations trigger re-render
    useRef,            // { el: HTMLElement } ref to DOM node or child component
    useEffect,         // (fn, deps) — side effect after render, cleanup on unmount
    onWillStart,       // async () — before first render; await allowed
    onMounted,         // () — after first DOM insertion
    onWillUpdateProps, // (nextProps) — before props update
    onWillRender,      // () — before each render
    onRendered,        // () — after each render (not committed to DOM yet)
    onPatched,         // () — after DOM patched
    onWillUnmount,     // () — before unmount, sync cleanup
    onWillDestroy,     // () — final cleanup
    onError,           // (error) — catch errors from children
} from "@odoo/owl";
```

**`useEffect` pattern:**

```javascript
setup() {
    this.state = useState({ query: '' });
    useEffect(
        () => {
            const handler = () => this.state.query = window.location.search;
            window.addEventListener('popstate', handler);
            return () => window.removeEventListener('popstate', handler);  // cleanup
        },
        () => []  // deps — empty = run once on mount
    );
}
```

---

## Custom Field Widget (v17+)

```javascript
/** @odoo-module **/

import { registry } from "@web/core/registry";
import { standardFieldProps } from "@web/views/fields/standard_field_props";
import { Component } from "@odoo/owl";
import { useService } from "@web/core/utils/hooks";

class ColoredCharField extends Component {
    static template = "my_module.ColoredCharField";
    static props = {
        ...standardFieldProps,
        // extra custom props from options=""
        highlightColor: { type: String, optional: true },
    };

    get value() {
        return this.props.record.data[this.props.name] || '';
    }

    get style() {
        const color = this.props.highlightColor || 'blue';
        return `color: ${color};`;
    }

    onInput(ev) {
        this.props.record.update({ [this.props.name]: ev.target.value });
    }
}

registry.category("fields").add("colored_char", {
    component: ColoredCharField,
    supportedTypes: ["char"],
    extractProps: ({ options }) => ({
        highlightColor: options.highlight_color,
    }),
});
```

```xml
<t t-name="my_module.ColoredCharField">
    <div t-att-style="style">
        <t t-if="props.readonly">
            <span t-esc="value"/>
        </t>
        <t t-else="">
            <input type="text" t-att-value="value" t-on-input="onInput"/>
        </t>
    </div>
</t>
```

Use in view: `<field name="name" widget="colored_char" options="{'highlight_color': 'red'}"/>`

---

## Services (v17+)

```javascript
/** @odoo-module **/
import { useService } from "@web/core/utils/hooks";

setup() {
    // ORM service — all DB operations
    this.orm = useService("orm");

    // Notification service
    this.notification = useService("notification");

    // Dialog service
    this.dialog = useService("dialog");

    // Action service — navigate/open views
    this.action = useService("action");

    // Router
    this.router = useService("router");

    // User service
    this.user = useService("user");

    // Company service
    this.company = useService("company");

    // RPC service (raw HTTP)
    this.rpc = useService("rpc");
}
```

### ORM Service Methods

```javascript
const orm = useService("orm");

// search_read
const records = await orm.searchRead(
    "my.model",
    [["state", "=", "draft"]],
    ["name", "state", "partner_id"],
    { limit: 10, offset: 0, order: "name asc", context: { lang: "fr_FR" } }
);

// create
const newId = await orm.create("my.model", [{ name: "New", state: "draft" }]);

// write
await orm.write("my.model", [1, 2, 3], { state: "done" });

// unlink
await orm.unlink("my.model", [1, 2]);

// call Python method
const result = await orm.call(
    "my.model",
    "action_confirm",
    [[recordId]],
    { context: { from_js: true } }
);

// name_search
const choices = await orm.nameSearch("res.partner", "John", [], { limit: 10 });

// read
const data = await orm.read("my.model", [1, 2], ["name", "state"]);

// search
const ids = await orm.search("my.model", [["state", "=", "draft"]], { limit: 100 });

// fields_get
const fields = await orm.fieldsGet("my.model", ["name", "type", "relation"]);
```

### Notification Service

```javascript
this.notification.add("Saved!", { type: "success", sticky: false });
this.notification.add("Error!", { type: "danger", sticky: true });
this.notification.add("Warning", { type: "warning" });
this.notification.add("Info", { type: "info" });
```

### Dialog Service

```javascript
import { ConfirmationDialog } from "@web/core/confirmation_dialog/confirmation_dialog";

this.dialog.add(ConfirmationDialog, {
    title: "Confirm Delete",
    body: "Are you sure? This cannot be undone.",
    confirm: async () => {
        await this.orm.unlink("my.model", [this.props.recordId]);
        this.action.doAction({ type: "ir.actions.act_window_close" });
    },
    cancel: () => {},
});

// Custom dialog
import { Dialog } from "@web/core/dialog/dialog";
this.dialog.add(Dialog, {
    title: "My Dialog",
    // slots via component
});
```

### Action Service

```javascript
// Open view
await this.action.doAction("my_module.action_my_model");

// Open form for specific record
await this.action.doAction({
    type: "ir.actions.act_window",
    res_model: "my.model",
    res_id: 42,
    view_mode: "form",
    views: [[false, "form"]],
    target: "current",  // current | new | fullscreen
});

// Close dialog
await this.action.doAction({ type: "ir.actions.act_window_close" });

// Reload
this.action.switchView("form");  // reload current view
```

---

## Patching Existing Components (v17+)

```javascript
/** @odoo-module **/
import { patch } from "@web/core/utils/patch";
import { FormController } from "@web/views/form/form_controller";
import { ListController } from "@web/views/list/list_controller";
import { KanbanController } from "@web/views/kanban/kanban_controller";

// Add method to FormController
patch(FormController.prototype, {
    async beforeSave() {
        console.log("Before save:", this.model.root.data);
        return super.beforeSave(...arguments);
    },
    async afterSave() {
        this.notification.add("Saved!", { type: "success" });
        return super.afterSave(...arguments);
    },
});

// Patch a computed property
patch(FormController.prototype, {
    get actionMenuItems() {
        const items = super.actionMenuItems;
        items.other.push({
            key: "my_action",
            description: "My Custom Action",
            callback: () => this.myCustomAction(),
        });
        return items;
    },
    myCustomAction() {
        // custom logic
    },
});
```

### Patching Field Components

```javascript
import { CharField } from "@web/views/fields/char/char_field";

patch(CharField.prototype, {
    get value() {
        const val = super.value;
        return val ? val.trim() : val;
    },
});
```

---

## Client Actions (v17+)

```javascript
/** @odoo-module **/
import { Component, onWillStart, useState } from "@odoo/owl";
import { registry } from "@web/core/registry";
import { useService } from "@web/core/utils/hooks";

class MyDashboard extends Component {
    static template = "my_module.MyDashboard";
    static props = {};  // client actions receive no props from framework

    setup() {
        this.orm = useService("orm");
        this.state = useState({ data: null, loading: true });
        onWillStart(() => this.loadDashboardData());
    }

    async loadDashboardData() {
        this.state.data = await this.orm.call("my.model", "get_dashboard_data", [[]]);
        this.state.loading = false;
    }
}

registry.category("actions").add("my_module.my_dashboard", MyDashboard);
```

```xml
<!-- Register in XML -->
<record id="action_my_dashboard" model="ir.actions.client">
    <field name="name">My Dashboard</field>
    <field name="tag">my_module.my_dashboard</field>
</record>
```

---

## View Extensions via JS (v17+)

### Add button to control panel

```javascript
/** @odoo-module **/
import { ListController } from "@web/views/list/list_controller";
import { patch } from "@web/core/utils/patch";
import { useService } from "@web/core/utils/hooks";

patch(ListController.prototype, {
    setup() {
        super.setup();
        this.myService = useService("notification");
    },
});
```

```xml
<!-- Override controller template to add button -->
<t t-name="my_module.ListControllerButtons" t-inherit="web.ListController" t-inherit-mode="extension">
    <xpath expr="//div[hasclass('o_list_buttons')]" position="inside">
        <button class="btn btn-secondary" t-on-click="myAction">My Action</button>
    </xpath>
</t>
```

---

## Asset Bundle Registration

```python
# __manifest__.py
'assets': {
    # Main backend bundle
    'web.assets_backend': [
        'my_module/static/src/js/my_component.js',
        'my_module/static/src/xml/my_component.xml',
        'my_module/static/src/css/my_style.scss',
    ],
    # Lazy loaded (on demand)
    'web.assets_backend_lazy': [
        'my_module/static/src/js/heavy_chart.js',
    ],
    # Website frontend
    'web.assets_frontend': [
        'my_module/static/src/js/website.js',
    ],
    # Override/remove core assets
    'web.assets_backend': [
        # Prepend (loads before other assets)
        ('prepend', 'my_module/static/src/css/override.scss'),
        # Remove a core file
        ('remove', 'web/static/src/views/fields/statusbar/statusbar_field.js'),
        # Replace a core file
        ('replace', 'web/static/src/views/fields/statusbar/statusbar_field.js',
                    'my_module/static/src/js/custom_statusbar.js'),
    ],
    # Tests
    'web.assets_tests': [
        'my_module/static/tests/**/*.js',
    ],
},
```

---

## QUnit Tests (v17+)

```javascript
/** @odoo-module **/
import { makeTestEnv } from "@web/../tests/helpers/mock_env";
import { getFixture, mount } from "@web/../tests/helpers/utils";
import { MyComponent } from "@my_module/js/my_component";

QUnit.module("my_module.MyComponent", () => {
    QUnit.test("renders correctly", async (assert) => {
        const env = await makeTestEnv();
        const target = getFixture();
        await mount(MyComponent, target, { env, props: { title: "Test" } });
        assert.containsOnce(target, ".o_my_component");
        assert.strictEqual(target.querySelector("h3").textContent, "Test");
    });
});
```

---

## Version-Specific Changes

### v17 Changes

- `/** @odoo-module **/` required on all JS files
- `t-on-event` kebab-case required (PascalCase removed)
- Legacy `Widget` class **deprecated** (legacy/ compat folder exists; avoid in new code)
- Legacy `AbstractField` **deprecated** → use `registry.category("fields")`
- `odoo.define(...)` **deprecated** (compat shim in module_loader.js; use ES modules)
- `core/bus.js` → `@web/core/bus_service`
- Action manager: `do_action` → `action.doAction`
- Dialog: `Dialog.alert/confirm` statics → `dialog` service
- `session.uid` → `user.userId` from user service
- `session.user_context` → `user.context` from user service

### v18 Changes

- `useComponent()` hook: access current component from inside composable hooks
- `reactive()` from OWL: create standalone reactive store
- `useChildSubEnv()`: pass sub-environment to children without Component wrapper
- `toRaw()`: get non-reactive version of reactive object (performance)
- `markRaw()`: mark object to never be made reactive
- New `useSpreadsheet` hooks (EE)
- `owl.App` constructor signature stabilized

### v19 Changes

- Native TypeScript (`.ts` files accepted in bundles without compilation step)
- `useAutoScroll(ref)` built-in hook
- `Component.components` static is now optional (auto-discovered from template)
- `t-portal` directive for rendering outside component tree (modals, tooltips)

---

## Deprecated/Removed JS APIs Table

| API | Version removed | Replacement |
|-----|----------------|-------------|
| `odoo.define(...)` | v17 deprecated | `/** @odoo-module **/` + ES imports (compat shim exists) |
| `Widget.extend(...)` | v17 deprecated | `class MyComp extends Component` |
| `AbstractField.extend(...)` | v17 deprecated | OWL Component + fields registry |
| `FieldChar`, `FieldMany2one` etc. | v17 | OWL field components from `@web/views/fields/` |
| `session.uid` | v17 | `useService("user").userId` |
| `session.user_context` | v17 | `useService("user").context` |
| `ajax.rpc()` | v17 | `useService("orm")` or `useService("rpc")` |
| `this.do_action()` | v17 | `useService("action").doAction()` |
| `Dialog.alert()` | v17 | `useService("dialog").add(AlertDialog, ...)` |
| `Dialog.confirm()` | v17 | `useService("dialog").add(ConfirmationDialog, ...)` |
| `willStart()` class method | OWL2/v16 | `onWillStart()` hook in `setup()` |
| `mounted()` class method | OWL2/v16 | `onMounted()` hook in `setup()` |
| `willUnmount()` class method | OWL2/v16 | `onWillUnmount()` hook in `setup()` |
| `patched()` class method | OWL2/v16 | `onPatched()` hook in `setup()` |
| `t-on-Click` (PascalCase) | OWL2/v16 | `t-on-click` (kebab) |
| `owl.mount()` direct | OWL2 | `new App(Component).mount(target)` |
| `this.trigger('event')` | OWL2 | `this.props.onEvent(...)` or bus |
| `QWeb.render(template, ctx)` | v17 | OWL template system |
| `field_registry.add()` | v17 | `registry.category("fields").add()` |
| `view_registry.add()` | v17 | `registry.category("views").add()` |
| `action_registry.add()` | v17 | `registry.category("actions").add()` |
| `component_registry.add()` | v17 | `registry.category("main_components").add()` |
| `archParseBoolean()` | v17 (from `@web/views/utils`) | v18 | `exprToBoolean()` from `@web/core/utils/strings` |

---

## Registry Categories Reference (source-verified)

| Category | Purpose | Usage |
|----------|---------|-------|
| `"fields"` | Field widgets (bound to model fields) | `<field name="x" widget="my_widget"/>` |
| `"view_widgets"` | Non-field view components | `<widget name="my_widget"/>` |
| `"views"` | View types (form/list/kanban/calendar/etc.) | `type="my_view"` in action |
| `"actions"` | Client action components | `<field name="tag">action_tag</field>` in `ir.actions.client` |
| `"services"` | Injectable services | `useService("my_service")` |
| `"main_components"` | Always-mounted root components | Systray, command palette |
| `"systray"` | Systray items (top-right nav icons) | `position: 0` (lower = right) |
| `"formatters"` | Value → display string | `formatFieldValue(val, field)` |
| `"parsers"` | String → typed value | Used on input change |
| `"dialogs"` | Named dialogs | `dialog.add("my_dialog", props)` |
| `"cogMenu"` | Action menu items (cog dropdown) | Form/list view |
| `"favoriteMenu"` | Search panel favorites items | SearchPanel |
| `"command_provider"` | Command palette providers | `ctrl+K` palette |
| `"debug"` | Debug menu items | Debug mode only |
| `"debug_section"` | Debug menu sections (v18+) | Debug mode only |
| `"effects"` | Visual effects (rainbow man, etc.) | `effect.add({type: "rainbow_man"})` |
| `"color_picker_tabs"` | Color picker tabs (v19 only) | Color picker widget |
| `"group_config_items"` | Group config menu items (v19 only) | List view group headers |
| `"public.interactions"` | Website public interactions (v19 only) | Website frontend |

---

## Complete Field Widget Pattern (source-verified v17/v18/v19)

### Full field widget with all hooks

```javascript
/** @odoo-module **/

import { _t } from "@web/core/l10n/translation";
import { registry } from "@web/core/registry";
import { standardFieldProps } from "@web/views/fields/standard_field_props";
import { useInputField } from "@web/views/fields/input_field_hook";
// v17: import { archParseBoolean } from "@web/views/utils";
// v18+: import { exprToBoolean } from "@web/core/utils/strings";
import { exprToBoolean } from "@web/core/utils/strings";   // v18/v19
import { formatChar } from "@web/views/fields/formatters";

import { Component, useRef } from "@odoo/owl";

export class MyCustomField extends Component {
    static template = "my_module.MyCustomField";
    static props = {
        ...standardFieldProps,           // id, name, readonly, record
        myOption: { type: Boolean, optional: true },
        placeholder: { type: String, optional: true },
    };
    static defaultProps = { myOption: false };

    setup() {
        this.inputRef = useRef("input");

        // useInputField: sync input↔record.data without losing user edits during onchange
        useInputField({
            getValue: () => this.props.record.data[this.props.name] || "",
            parse: (v) => v.trim(),
            refName: "input",         // matches t-ref="input" in template
        });
    }

    // Read field value (used in readonly mode)
    get formattedValue() {
        return formatChar(this.props.record.data[this.props.name]);
    }

    // Read field metadata
    get fieldInfo() {
        return this.props.record.fields[this.props.name];
    }

    get isRequired() {
        return this.props.record.isRequired(this.props.name);
    }

    // Write field value
    async updateValue(newValue) {
        await this.props.record.update({ [this.props.name]: newValue });
    }
}

// Field descriptor object — registered, NOT the class directly
export const myCustomField = {
    component: MyCustomField,
    displayName: _t("My Custom Field"),
    supportedTypes: ["char"],          // which Odoo field types accept this widget

    // "supportedOptions" documents widget options for Settings UI (v17+)
    supportedOptions: [
        {
            label: _t("My Option"),
            name: "my_option",
            type: "boolean",
            help: _t("Enable to activate special behavior."),
        },
    ],

    // extractProps: maps arch attrs + options to component props
    // v17/v18: ({ attrs, options }) => ({...})
    // v19: ({ attrs, options, placeholder }) => ({...})  ← placeholder extracted by framework
    extractProps: ({ attrs, options, placeholder }) => ({
        myOption: exprToBoolean(options.my_option || "false"),
        placeholder,                   // v19: extracted automatically from arch
        // placeholder: attrs.placeholder,  ← v17/v18 pattern
    }),
};

registry.category("fields").add("my_custom_field", myCustomField);
```

```xml
<!-- my_module/static/src/xml/my_custom_field.xml -->
<templates>
    <t t-name="my_module.MyCustomField">
        <t t-if="props.readonly">
            <span t-esc="formattedValue" class="o_field_widget"/>
        </t>
        <t t-else="">
            <input
                t-ref="input"
                type="text"
                class="o_input"
                t-att-placeholder="props.placeholder"
                t-att-id="props.id"
            />
        </t>
    </t>
</templates>
```

Use in view: `<field name="my_char" widget="my_custom_field" options="{'my_option': true}"/>`

### `archParseBoolean` → `exprToBoolean` change (v17 → v18+)

```javascript
// ❌ v17 only — from @web/views/utils
import { archParseBoolean } from "@web/views/utils";
const val = archParseBoolean(attrs.password);  // "true"/"false"/""

// ✅ v18/v19 — from @web/core/utils/strings
import { exprToBoolean } from "@web/core/utils/strings";
const val = exprToBoolean(options.my_bool || "false");
```

### `extractProps` placeholder change (v19)

```javascript
// v17/v18: placeholder from attrs
extractProps: ({ attrs, options }) => ({
    placeholder: attrs.placeholder,
})

// v19: placeholder extracted by framework and passed as 3rd named param
extractProps: ({ attrs, options, placeholder }) => ({
    placeholder,   // pre-extracted from arch
})
```

### `CharField` `supportedTypes` change (v19)

```javascript
// v17/v18
supportedTypes: ["char"]

// v19: CharField now also handles text type
supportedTypes: ["char", "text"]
```

### `useRecordObserver` — React to record changes in a field (v18+)

```javascript
import { useRecordObserver } from "@web/model/relational_model/utils";

// Inside setup():
// Fires whenever the record's data relevant to this field changes
// Useful for async side effects (fetching additional info based on field value)
setup() {
    useRecordObserver(async (record, props) => {
        const value = record.data[props.name];
        // run async work when field value changes
        this.state.preview = value ? await this.fetchPreview(value) : null;
    });
}
```

`useRecordObserver` is NOT available in v17. Use `onWillUpdateProps` + `useEffect` combo instead.

### Custom formatter/parser (for custom display formatting)

```javascript
import { registry } from "@web/core/registry";

// Register a custom formatter for a field type
registry.category("formatters").add("my_type", (value, options) => {
    if (!value) return "";
    return `${value} (custom)`;
});

// Register a custom parser (string input → typed value)
registry.category("parsers").add("my_type", (value, options) => {
    const parsed = parseFloat(value);
    if (isNaN(parsed)) throw new Error("Invalid value");
    return parsed;
});
```

---

## View Widget (`view_widgets` registry) — NOT a field widget

Use `<widget name="..."/>` in XML (no model field binding required):

```javascript
/** @odoo-module **/

import { registry } from "@web/core/registry";
import { standardWidgetProps } from "@web/views/widgets/standard_widget_props";
import { useService } from "@web/core/utils/hooks";
import { Component } from "@odoo/owl";

export class MyViewWidget extends Component {
    static template = "my_module.MyViewWidget";
    static props = {
        ...standardWidgetProps,   // readonly, record
        // extra custom props:
        label: { type: String, optional: true },
        modelField: { type: String, optional: true },
    };

    setup() {
        this.orm = useService("orm");
        this.notification = useService("notification");
    }

    get currentValue() {
        return this.props.modelField
            ? this.props.record.data[this.props.modelField]
            : null;
    }

    async onClick() {
        await this.orm.call(
            this.props.record.resModel,
            "my_button_method",
            [[this.props.record.resId]]
        );
        this.notification.add("Done!", { type: "success" });
        await this.props.record.load();
    }
}

// standardWidgetProps = { readonly: Boolean (optional), record: Object }
registry.category("view_widgets").add("my_view_widget", {
    component: MyViewWidget,
    extractProps: ({ attrs }) => ({
        label: attrs.label,
        modelField: attrs.model_field,
    }),
});
```

```xml
<!-- In a form view -->
<widget name="my_view_widget" label="My Label" model_field="partner_id"/>
```

`standardWidgetProps` — same in v17/v18/v19: `{ readonly, record }`.

### `field` vs `view_widgets` vs `main_components`

| | `fields` | `view_widgets` | `main_components` |
|---|---|---|---|
| XML tag | `<field name="x" widget="y"/>` | `<widget name="y"/>` | no XML — always mounted |
| Bound to field? | ✅ yes | ❌ no | ❌ no |
| `standardFieldProps` | ✅ | ❌ | ❌ |
| `standardWidgetProps` | ❌ | ✅ | ❌ |
| `record` prop | ✅ (from `standardFieldProps`) | ✅ (from `standardWidgetProps`) | ❌ |

---

## Custom View Type (v17+)

Full custom view: ArchParser + Controller + Model + Renderer:

```javascript
/** @odoo-module **/
// my_module/static/src/views/my_view/my_view.js

import { registry } from "@web/core/registry";
import { Component, useState, onWillStart } from "@odoo/owl";
import { useService } from "@web/core/utils/hooks";
import { standardViewProps } from "@web/views/standard_view_props";

// 1. ArchParser — parses the <view arch> XML
class MyViewArchParser {
    parse(arch, models, modelName) {
        const archInfo = { fields: {}, myOption: false };
        // parse arch.firstChild for custom attributes
        const archNode = arch;
        archInfo.myOption = archNode.getAttribute("my_option") === "1";
        return archInfo;
    }
}

// 2. Model — data loading / manipulation
class MyViewModel {
    constructor(env, { orm, resModel, fields, activeFields }) {
        this.orm = orm;
        this.resModel = resModel;
        this.fields = fields;
    }

    async load(params) {
        this.records = await this.orm.searchRead(
            this.resModel,
            params.domain || [],
            Object.keys(this.fields),
            { limit: params.limit || 80 }
        );
        return this.records;
    }
}

// 3. Renderer — displays the data
class MyViewRenderer extends Component {
    static template = "my_module.MyViewRenderer";
    static props = {
        records: { type: Array },
        archInfo: { type: Object },
        openRecord: { type: Function, optional: true },
    };
}

// 4. Controller — orchestrates Model + Renderer + control panel
class MyViewController extends Component {
    static template = "my_module.MyViewController";
    static props = {
        ...standardViewProps,
        Model: { type: Function },
        Renderer: { type: Function },
        archInfo: { type: Object },
    };
    static components = { MyViewRenderer };

    setup() {
        this.orm = useService("orm");
        this.state = useState({ records: [] });

        const { Model } = this.props;
        this.model = new Model(this.env, {
            orm: this.orm,
            resModel: this.props.resModel,
            fields: this.props.fields,
            activeFields: this.props.archInfo.activeFields,
        });

        onWillStart(async () => {
            this.state.records = await this.model.load({
                domain: this.props.domain,
            });
        });
    }

    async openRecord(record) {
        await this.env.services.action.doAction({
            type: "ir.actions.act_window",
            res_model: this.props.resModel,
            res_id: record.id,
            views: [[false, "form"]],
            target: "current",
        });
    }
}

// 5. Register the view
export const myView = {
    type: "my_view",
    display_name: "My Custom View",
    icon: "fa fa-th",
    multiRecord: true,
    searchMenuTypes: ["filter", "groupBy", "favorite"],
    ArchParser: MyViewArchParser,
    Controller: MyViewController,
    Model: MyViewModel,
    Renderer: MyViewRenderer,
    props: (genericProps, view) => {
        const archInfo = new view.ArchParser().parse(
            genericProps.arch,
            genericProps.relatedModels,
            genericProps.resModel
        );
        return {
            ...genericProps,
            Model: view.Model,
            Renderer: view.Renderer,
            archInfo,
        };
    },
};

registry.category("views").add("my_view", myView);
```

```xml
<!-- Register in Python (ir.actions.act_window)  -->
<record id="action_my_view" model="ir.actions.act_window">
    <field name="name">My Records</field>
    <field name="res_model">my.model</field>
    <field name="view_mode">my_view,form</field>
</record>
```

---

## `doAction` Complete Options Reference (v17/v18/v19 same)

```javascript
const action = useService("action");

await action.doAction(actionRequest, {
    // Context merged into action's context
    additionalContext: { default_state: "draft" },

    // Callback when dialog/secondary action closes
    onClose: async () => {
        await this.props.record.load();
    },

    // If true, clears all breadcrumbs (starts fresh navigation)
    clearBreadcrumbs: false,

    // Breadcrumb position: "new" (default), "replace" (replace last), "current"
    stackPosition: "new",
});

// Open a record in a dialog (new window)
await action.doAction({
    type: "ir.actions.act_window",
    res_model: "my.model",
    res_id: 42,
    views: [[false, "form"]],
    target: "new",          // "new" = dialog, "current" = same tab, "fullscreen"
    context: { default_partner_id: this.props.record.resId },
}, {
    onClose: async () => {
        // refresh after dialog closes
        await this.props.record.load();
    },
});

// Open URL action
await action.doAction({
    type: "ir.actions.act_url",
    url: "https://example.com",
    target: "new",          // "new" = new tab, "self" = same tab
});

// Call server action
await action.doAction({
    type: "ir.actions.server",
    id: serverActionId,     // or use xml id: "my_module.my_server_action"
}, {
    additionalContext: { active_ids: selectedIds },
});

// Close current dialog
await action.doAction({ type: "ir.actions.act_window_close" });
await action.doAction({ type: "ir.actions.act_window_close" }, {
    onClose: () => { /* optional callback */ }
});

// Navigate to an existing action by XML id
await action.doAction("my_module.action_my_view");
```

---

## OWL Advanced Patterns (v17/v18/v19)

### `reactive()` — standalone reactive store (v18+ from `@odoo/owl`)

```javascript
import { reactive, Component, useEffect } from "@odoo/owl";

// Create a reactive object OUTSIDE a component (shared store pattern)
const globalState = reactive({ count: 0, items: [] });

class MyComponent extends Component {
    setup() {
        // Component will re-render when globalState changes
        this.state = reactive(globalState);

        useEffect(() => {
            // runs after render
            console.log("count changed to", this.state.count);
        }, () => [this.state.count]);  // deps function
    }
}
```

`reactive()` is available in v17 but rarely used publicly. v18+ relies on it internally for services.

### `markRaw()` / `toRaw()` — performance optimizations (v18+)

```javascript
import { markRaw, toRaw } from "@odoo/owl";

// markRaw: prevent object from being made reactive (saves memory/perf)
// Use for large non-reactive objects (e.g., third-party libraries, big datasets)
class MyComponent extends Component {
    setup() {
        this.chart = markRaw(null);  // chart instance won't be proxied
        onMounted(() => {
            this.chart = markRaw(new SomeBigChartLibrary(this.chartRef.el));
        });
    }
}

// toRaw: get the non-reactive underlying object
const rawData = toRaw(this.state.data);   // bypass reactivity for read
```

### `t-portal` — render outside component tree (v19 ONLY)

```xml
<!-- Renders at document.body level, not inside current component DOM -->
<t t-portal="'body'">
    <div class="floating-tooltip" t-att-style="tooltipStyle">
        <t t-esc="tooltipText"/>
    </div>
</t>

<!-- Any valid CSS selector or DOM element ref works -->
<t t-portal="'#my-container'">
    <MyModal/>
</t>
```

### `t-slot` / `t-set-slot` — slot system (all versions)

```xml
<!-- Parent template defining slots -->
<t t-name="my_module.Panel">
    <div class="panel">
        <div class="panel-header">
            <t t-slot="header">Default Header</t>  <!-- slot with default -->
        </div>
        <div class="panel-body">
            <t t-slot="default"/>   <!-- unnamed/default slot -->
        </div>
        <div class="panel-footer">
            <t t-slot="footer"/>
        </div>
    </div>
</t>
```

```xml
<!-- Consumer template filling slots -->
<Panel>
    <t t-set-slot="header">
        <h2>Custom Header</h2>
    </t>
    <!-- default slot content directly between tags -->
    <p>Main body content here</p>
    <t t-set-slot="footer">
        <button>Close</button>
    </t>
</Panel>
```

### OWL template inheritance `t-inherit` (v17+)

Used to extend/override OWL templates from other modules:

```xml
<!-- Extension mode: add to existing template -->
<t t-name="my_module.CharField"
   t-inherit="web.CharField"
   t-inherit-mode="extension">
    <xpath expr="//input" position="before">
        <span class="prefix">PRE-</span>
    </xpath>
    <xpath expr="//span[@class='o_field_widget']" position="after">
        <small class="suffix">validated</small>
    </xpath>
</t>

<!-- Primary mode: completely replace template -->
<t t-name="my_module.FormView"
   t-inherit="web.FormView"
   t-inherit-mode="primary">
    <xpath expr="//div[hasclass('o_form_view')]" position="attributes">
        <attribute name="class" add="my_custom_class"/>
    </xpath>
</t>
```

XPath `position` values: `inside`, `before`, `after`, `replace`, `attributes`.

### `useSubEnv()` / `useChildSubEnv()` — provide environment data

```javascript
import { useSubEnv, useChildSubEnv } from "@odoo/owl";

// useSubEnv: adds to env for THIS component AND all children
class ParentWithContext extends Component {
    setup() {
        useSubEnv({
            myService: new MyService(),
            currentUser: this.props.userId,
        });
        // this.env.myService now available here + all children
    }
}

// useChildSubEnv: adds to env for CHILDREN ONLY (not self)
class DataProvider extends Component {
    setup() {
        useChildSubEnv({
            recordId: this.props.id,
        });
        // this.env.recordId NOT available in DataProvider itself
        // but IS available in all child components
    }
}
```

### `useExternalListener` — DOM events outside component (v17+)

```javascript
import { useExternalListener } from "@odoo/owl";

// Attaches event to document/window — auto-removed on unmount
setup() {
    useExternalListener(window, "resize", this.onWindowResize.bind(this));
    useExternalListener(document, "keydown", this.onKeydown.bind(this));
    useExternalListener(document.body, "click", this.onOutsideClick.bind(this));
}
```

### `useBus` — subscribe to an EventBus (v17+)

```javascript
import { useBus } from "@web/core/utils/hooks";

setup() {
    // auto-unsubscribes on unmount
    useBus(this.env.bus, "MY_EVENT", (ev) => {
        console.log("received:", ev.detail);
    });

    // Trigger event
    this.env.bus.trigger("MY_EVENT", { data: 42 });
}
```

### Systray item (v17+)

```javascript
import { registry } from "@web/core/registry";
import { Component } from "@odoo/owl";

class MyNotificationSystray extends Component {
    static template = "my_module.MySystrayItem";
    static props = {};
}

registry.category("systray").add("my_module.MyNotificationSystray", {
    Component: MyNotificationSystray,
    sequence: 10,   // lower = more to the right in systray
});
```

---

## Translation in JS (v17+)

```javascript
import { _t } from "@web/core/l10n/translation";

// Static string — translated at call time
const msg = _t("Hello World");

// With variables — use template literal or concatenation  
// ❌ DON'T pass expressions to _t — not extractable
const bad = _t(`Hello ${name}`);

// ✅ DO: compose after translation
const good = _t("Hello %s").replace("%s", name);
// Or use markup for HTML
import { markup } from "@odoo/owl";
const html = markup(_t("<b>Bold text</b>"));
```

---

## Record API in Field Widgets (source-verified)

```javascript
// Access field value (read)
const value = this.props.record.data[this.props.name];

// Read field metadata
const field = this.props.record.fields[this.props.name];
// field.type → "char", "many2one", etc.
// field.string → display label
// field.required → boolean
// field.readonly → boolean
// field.relation → "res.partner" (for relational fields)
// field.domain → domain constraint
// field.size → max chars (char fields)
// field.trim → auto-trim (char fields)
// field.translate → translatable (char fields)

// Update field value (write) — triggers onchange
await this.props.record.update({ [this.props.name]: newValue });

// Reload record from DB
await this.props.record.load();

// Check field validity
const isInvalid = this.props.record.isFieldInvalid(this.props.name);
const isRequired = this.props.record.isRequired(this.props.name);

// Reset field validity state
this.props.record.resetFieldValidity(this.props.name);

// Model/res info
const resModel = this.props.record.resModel;  // e.g., "sale.order"
const resId = this.props.record.resId;         // record id or null (new)
const isNew = this.props.record.isNew;          // true if unsaved

// For Many2one: value is [id, display_name] tuple or false
const [id, name] = this.props.record.data["partner_id"] || [];
```
