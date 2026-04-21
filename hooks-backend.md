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

#### `wp3dconf/admin/settings/section`

Fires inside the plugin settings page, allowing addons to register additional settings sections.

```php
add_action( 'wp3dconf/admin/settings/section', function() {
  ?>
  <div class="wp3dconf-settings-section">
    <h2><?php esc_html_e( 'My Addon Settings', 'my-addon' ); ?></h2>
    <table class="form-table">
      <tr>
        <th><?php esc_html_e( 'My Option', 'my-addon' ); ?></th>
        <td>
          <input type="text" name="my_addon_option"
            value="<?php echo esc_attr( get_option( 'my_addon_option' ) ); ?>">
        </td>
      </tr>
    </table>
  </div>
  <?php
} );
```

---

#### `wp3dconf/admin/settings/save`

Fires when the plugin settings form is submitted. Use this to save additional addon settings.

```php
add_action( 'wp3dconf/admin/settings/save', function() {
  if ( isset( $_POST['my_addon_option'] ) ) {
    update_option( 'my_addon_option', sanitize_text_field( wp_unslash( $_POST['my_addon_option'] ) ) );
  }
} );
```

---

#### `wp3dconf/admin/configurator/metabox`

Fires inside the configurator edit screen, below the main editor. Use this to add a custom metabox or additional UI.

| Parameter  | Type | Description |
|------------|------|-------------|
| `$post_id` | int  | The configurator post ID |

```php
add_action( 'wp3dconf/admin/configurator/metabox', function( $post_id ) {
  ?>
  <div class="postbox">
    <h2><?php esc_html_e( 'My Addon', 'my-addon' ); ?></h2>
    <div class="inside">
      <!-- Custom metabox content -->
    </div>
  </div>
  <?php
}, 10, 1 );
```

---

## PHP Admin Filters

#### `wp3dconf/admin/configurator/tabs`

Filters the tab list in the configurator editor sidebar. Use this to add custom tabs.

| Parameter | Type  | Description |
|-----------|-------|-------------|
| `$tabs`   | Array | Registered tabs keyed by tab ID |

```php
add_filter( 'wp3dconf/admin/configurator/tabs', function( $tabs ) {
  $tabs['my-tab'] = array(
    'label' => __( 'My Tab', 'my-addon' ),
    'icon'  => 'dashicons-admin-generic',
  );
  return $tabs;
} );
```

---

#### `wp3dconf/admin/layer/types`

Filters the available layer types. Use this to register a custom layer type.

| Parameter | Type  | Description |
|-----------|-------|-------------|
| `$types`  | Array | Registered layer types keyed by type slug |

```php
add_filter( 'wp3dconf/admin/layer/types', function( $types ) {
  $types['my-type'] = array(
    'label' => __( 'My Layer Type', 'my-addon' ),
  );
  return $types;
} );
```