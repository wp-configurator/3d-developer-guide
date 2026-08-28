# Backend Hooks & Filters

Actions and filters available in the WordPress admin - covering the configurator editor and general admin/settings areas.

---

## JavaScript - Lifecycle Events

Before registering editor hooks, wait for the appropriate event.

| Event | Description |
|-------|-------------|
| `wp3dconf/editor:before-mount` | Fires before the Vue editor app is mounted. `event.detail.editor` exposes the editor instance. Use this to register custom Vue components. |
| `wp3dconf/editor:ready` | Fires when the editor is fully initialised. Register all editor actions and filters here. |

```js
// Register custom Vue components before the editor mounts
window.addEventListener('wp3dconf/editor:before-mount', function(event) {
  const editor = event.detail.editor;

  editor.component('MyComponent', MyComponent);
});

// Register hooks once the editor is ready
window.addEventListener('wp3dconf/editor:ready', function(event) {
  const { addAction, addFilter } = window.WP3dConf;

  addAction('wp3dconf/editor/layer-added', function({ layer }) {
    console.log('New layer added:', layer);
  }, 10);
});
```

---

## JavaScript - Global API

```js
const { addAction, addFilter, hasAction, removeAction } = window.WP3dConf;
```

### `addAction( hook, callback, priority )`

| Parameter  | Type     | Default | Description |
|------------|----------|---------|-------------|
| `hook`     | String   | -       | The hook name |
| `callback` | Function | -       | The function to call |
| `priority` | Number   | `10`    | Lower numbers run first |

### `addFilter( hook, callback, priority )`

Same signature as `addAction`. The `callback` must return the first argument (the filtered value).

---

## JavaScript Editor Actions

#### `wp3dconf/editor/model-loaded`

Fires after the 3D model has been loaded in the editor canvas.

| Parameter | Type   | Description |
|-----------|--------|-------------|
| `store`   | Object | The editor Pinia store |
| `store3D` | Object | The Three.js/TresJS store |

---

#### `wp3dconf/editor/configurator-saved`

Fires after the configurator has been successfully saved.

| Parameter | Type   | Description |
|-----------|--------|-------------|
| `store`   | Object | The editor Pinia store |

```js
window.addEventListener('wp3dconf/editor:ready', function() {
  const { addAction } = window.WP3dConf;

  addAction('wp3dconf/editor/configurator-saved', function({ store }) {
    console.log('Saved. Layers:', store.layers);
  }, 10);
});
```

---

#### `wp3dconf/editor/layer-inserted`

Fires when a new layer is inserted into the layer tree (e.g. via drag-and-drop).

| Parameter | Type   | Description |
|-----------|--------|-------------|
| `store`   | Object | The editor Pinia store |
| `layer`   | Object | The inserted layer |
| `parent`  | Object | The parent layer or group |

---

#### `wp3dconf/editor/layer-added`

Fires when a new layer is added via the Add button.

| Parameter | Type   | Description |
|-----------|--------|-------------|
| `store`   | Object | The editor Pinia store |
| `layer`   | Object | The newly added layer |

```js
window.addEventListener('wp3dconf/editor:ready', function() {
  const { addAction } = window.WP3dConf;

  addAction('wp3dconf/editor/layer-added', function({ layer }) {
    console.log('New layer added:', layer);
  }, 10);
});
```

---

#### `wp3dconf/editor/layer-deleted`

Fires when a layer is deleted from the layer tree.

| Parameter | Type   | Description |
|-----------|--------|-------------|
| `store`   | Object | The editor Pinia store |
| `layer`   | Object | The deleted layer |
| `parent`  | Object | The parent layer or group |

---

#### `wp3dconf/editor/layer-duplicated`

Fires when a layer is duplicated.

| Parameter  | Type   | Description |
|------------|--------|-------------|
| `store`    | Object | The editor Pinia store |
| `layer`    | Object | The original layer |
| `parent`   | Object | The parent layer or group |
| `newLayer` | Object | The newly created duplicate |

---

#### `wp3dconf/editor/layer-reordered`

Fires after layers have been reordered in the layer tree.

| Parameter | Type   | Description |
|-----------|--------|-------------|
| `store`   | Object | The editor Pinia store |

```js
addAction('wp3dconf/editor/layer-reordered', function({ store }) {
  console.log('Reordered layers:', store.layers);
}, 10);
```

---

#### `wp3dconf/editor/configurator-save-failed`

Fires when a save request fails (network error or a rejected response from the server).

| Parameter | Type   | Description |
|-----------|--------|-------------|
| `store`   | Object | The editor Pinia store |
| `error`   | Mixed  | The error payload returned by the failed request |

```js
addAction('wp3dconf/editor/configurator-save-failed', function({ store, error }) {
  console.error('Save failed:', error);
}, 10);
```

---

#### `wp3dconf/editor/trigger-save`

The host listens for this action to trigger a save programmatically — it's the mechanism `createHostSaveSync().triggerSave()` from `wp3dconf-module-kit` rides on (see [`extend-modules.md`](extend-modules.md)). You won't normally add a listener to this one yourself; call `doAction` to ask the host to save on your module's behalf.

```js
window.WP3dConf.doAction('wp3dconf/editor/trigger-save');
```

---

## JavaScript Editor Filters

#### `wp3dconf/editor/configurator-data`

Filters the `FormData` object before it is submitted to the server on save. Use this to append additional fields.

| Parameter  | Type     | Description |
|------------|----------|-------------|
| `formData` | FormData | The form data object being submitted |
| `store`    | Object   | The editor Pinia store |

```js
addFilter('wp3dconf/editor/configurator-data', function(formData, { store }) {
  formData.append('my_custom_field', 'my_value');
  return formData;
}, 10);
```

---

#### `wp3dconf/editor/layer-classes`

Filters the CSS classes applied to a layer row in the editor layer panel.

| Parameter | Type   | Description |
|-----------|--------|-------------|
| `classes` | Array  | Array of CSS class strings |
| `store`   | Object | The editor Pinia store |

```js
window.addEventListener('wp3dconf/editor:ready', function() {
  const { addFilter } = window.WP3dConf;

  addFilter('wp3dconf/editor/layer-classes', function(classes, { store }) {
    classes.push('custom-class');
    return classes;
  }, 10);
});
```

---

## PHP Editor Actions

#### `wp3dconf/editor/after_enqueue`

Fires after all editor scripts and styles have been enqueued. Use this to enqueue additional assets that depend on the editor.

```php
add_action( 'wp3dconf/editor/after_enqueue', function() {
  wp_enqueue_script( 'my-editor-addon', plugin_dir_url( __FILE__ ) . 'editor-addon.js' );
} );
```

---

## PHP Admin Actions

#### `wp3dconf/admin/register_post_types`

Fires when the plugin registers its post types. Use this to register additional post types alongside `wp3dconf`.

```php
add_action( 'wp3dconf/admin/register_post_types', function() {
  register_post_type( 'my_addon_type', array( /* ... */ ) );
} );
```

---

#### `wp3dconf/admin/register_submenus`

Fires while the plugin registers its wp-admin submenu pages (under `edit.php?post_type=wp3dconf`). Use this to add your own submenu page.

```php
add_action( 'wp3dconf/admin/register_submenus', function() {
  add_submenu_page(
    'edit.php?post_type=wp3dconf',
    __( 'My Addon', 'my-addon' ),
    __( 'My Addon', 'my-addon' ),
    'manage_options',
    'my-addon-page',
    'my_addon_render_page'
  );
} );
```

---

## PHP Admin Filters

> If you're building a module rather than a standalone addon, prefer overriding
> `layer_settings_controls()`, `layer_types()`, `global_settings_controls()`,
> or `toolbar_icon()` on your `Module_Base` subclass instead of hooking these
> filters directly — see [`extend-modules.md`](extend-modules.md). They exist
> as plain WordPress filters too because `Module_Base` itself is built on top
> of them.

#### `wp3dconf/admin/settings_fields`

Filters the tab/field definitions on the **wp3dconf → Settings** page. Unlike a typical settings-page hook, this isn't an action that echoes HTML — it's a filter over the same array-based field schema the core settings themselves are defined with, keyed by tab label.

| Parameter | Type  | Description |
|-----------|-------|-------------|
| `$fields` | Array | Tab label => list of field definitions |

```php
add_filter( 'wp3dconf/admin/settings_fields', function( $fields ) {
  $fields[ __( 'My Addon', 'my-addon' ) ] = array(
    array(
      'id'      => 'my_addon_option',
      'label'   => __( 'My Option', 'my-addon' ),
      'type'    => 'text',
      'default' => '',
    ),
  );
  return $fields;
} );
```

---

#### `wp3dconf/editor/tabs`

Filters the tab list in the configurator editor sidebar (default tabs: `model_structure`, `global_settings`).

| Parameter | Type  | Description |
|-----------|-------|-------------|
| `$tabs`   | Array | Registered tabs keyed by tab ID, each `['name' => ..., 'icon' => ...]` |

```php
add_filter( 'wp3dconf/editor/tabs', function( $tabs ) {
  $tabs['my-tab'] = array(
    'name' => __( 'My Tab', 'my-addon' ),
    'icon' => 'settings',
  );
  return $tabs;
} );
```

---

#### `wp3dconf/editor/layer_types`

Filters the available layer types shown in the "Model Structure" tab's draggable layer list (default types: `group`, `subgroup`, `option`). This is what `Module_Base::layer_type()` / `layer_types()` writes into on a module's behalf.

| Parameter | Type  | Description |
|-----------|-------|-------------|
| `$types`  | Array | Registered layer types keyed by type slug, each `['name' => ..., 'icon' => ...]` |

```php
add_filter( 'wp3dconf/editor/layer_types', function( $types ) {
  $types['my-type'] = array(
    'name' => __( 'My Layer Type', 'my-addon' ),
    'icon' => 'edit',
  );
  return $types;
} );
```

---

#### `wp3dconf/editor/global_settings_controls`

Filters the editor's global-settings schema (the "Global Settings" tab). This is what `Module_Base::global_settings_controls()` writes into on a module's behalf — see `modules/woocommerce/class-woocommerce.php` for a worked example of adding an option to an existing control vs. a whole new section.

| Parameter  | Type  | Description |
|------------|-------|-------------|
| `$settings`| Array | Settings schema, keyed by section |

---

#### `wp3dconf/editor/layer_settings_controls`

Filters the schema for the per-layer "Settings" panel shown when a layer is selected in the editor (price, sale price, description, CSS class, etc. live here by default). This is what `Module_Base::layer_settings_controls()` writes into on a module's behalf — see `modules/custom-fields/class-custom-fields.php`'s `layer_settings_controls()` for an example that adds a whole new "Field Settings" section, gated to only show for its own layer types.

| Parameter  | Type  | Description |
|------------|-------|-------------|
| `$settings`| Array | Settings schema, keyed by section |

```php
add_filter( 'wp3dconf/editor/layer_settings_controls', function( $settings ) {
  $settings['my_section'] = array(
    'name'     => __( 'My Settings', 'my-addon' ),
    'settings' => array(
      'my_option' => array(
        'name' => __( 'My Option', 'my-addon' ),
        'type' => 'text',
      ),
    ),
  );
  return $settings;
} );
```