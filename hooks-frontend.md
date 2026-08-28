# Frontend Hooks & Filters

Actions and filters available on the customer-facing configurator page - both JavaScript (Alpine.js) and PHP.

---

## JavaScript - Lifecycle Event

Before registering hooks, wait for the frontend to be ready.

| Event | Description |
|-------|-------------|
| `wp3dconf/frontend:ready` | Fires when the Alpine.js frontend is fully initialised. Register all frontend actions and filters here. |

```js
window.addEventListener('wp3dconf/frontend:ready', function(event) {
  const { addAction, addFilter } = window.WP3dConf;

  addAction('wp3dconf/frontend/storeInitialized', function() {
    console.log('Frontend store ready');
  }, 10);
});
```

---

## JavaScript - Global API

All JavaScript hooks are accessed via `window.WP3dConf`.

```js
const { addAction, addFilter, hasAction, removeAction } = window.WP3dConf;
```

A utilities object is also available via `window.WP3dConfUtils`.

```js
const { log } = window.WP3dConfUtils;
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

## JavaScript Actions

#### `wp3dconf/frontend/storeInitialized`

Fires when the Alpine.js store has been initialised. No parameters are passed.

```js
window.addEventListener('wp3dconf/frontend:ready', function() {
  const { addAction } = window.WP3dConf;
  const { log } = window.WP3dConfUtils;

  addAction('wp3dconf/frontend/storeInitialized', function() {
    log('Store initialized');
  }, 10);
});
```

---

#### `wp3dconf/frontend/data/initialized`

Fires after the configurator data has been loaded into the store.

| Parameter | Type   | Description |
|-----------|--------|-------------|
| `store`   | Object | The Alpine.js store instance |

---

#### `wp3dconf/frontend/layer/activated`

Fires when a layer option is activated by the customer.

| Parameter | Type   | Description |
|-----------|--------|-------------|
| `uid`     | String | UID of the activated layer |
| `layer`   | Object | The layer object |
| `store`   | Object | The Alpine.js store instance |

---

#### `wp3dconf/frontend/layer/deactivated`

Fires when a layer option is deactivated.

| Parameter | Type   | Description |
|-----------|--------|-------------|
| `uid`     | String | UID of the deactivated layer |
| `layer`   | Object | The layer object |
| `store`   | Object | The Alpine.js store instance |

---

#### `wp3dconf/frontend/data/priceUpdated`

Fires whenever the calculated price is updated.

| Parameter | Type   | Description |
|-----------|--------|-------------|
| `store`   | Object | The Alpine.js store instance |

```js
addAction('wp3dconf/frontend/data/priceUpdated', function({ store }) {
  console.log('Updated price:', store.price);
}, 10);
```

---

#### `wp3dconf/frontend/layer/enabled` / `wp3dconf/frontend/layer/disabled`

Fire when a layer's enabled state changes (independent of selection/visibility — a disabled layer can't be clicked, e.g. via a Conditional Logic `set_state` action).

| Parameter | Type   | Description |
|-----------|--------|-------------|
| `uid`     | String | UID of the layer |
| `layer`   | Object | The layer object |
| `store`   | Object | The Alpine.js store instance |

---

#### `wp3dconf/frontend/layer/shown` / `wp3dconf/frontend/layer/hidden`

Fire when a layer's visibility changes.

| Parameter | Type   | Description |
|-----------|--------|-------------|
| `uid`     | String | UID of the layer |
| `layer`   | Object | The layer object |
| `store`   | Object | The Alpine.js store instance |

---

#### `wp3dconf/frontend/group/opened` / `wp3dconf/frontend/group/closed`

Fire when a group/subgroup layer is expanded or collapsed in the controls UI.

| Parameter | Type   | Description |
|-----------|--------|-------------|
| `uid`     | String | UID of the group layer |
| `layer`   | Object | The layer object |
| `store`   | Object | The Alpine.js store instance |

---

#### `wp3dconf/frontend/modelLoaded` / `wp3dconf/frontend/afterModelLoaded`

Fire once the 3D model has finished loading in the canvas. `modelLoaded` fires first and is used internally to apply the initial active-layer selection to the canvas; `afterModelLoaded` fires once that's done, so it's the safer hook for addon code that needs the canvas fully settled.

| Parameter | Type   | Description |
|-----------|--------|-------------|
| `store`   | Object | The Alpine.js store instance |

```js
addAction('wp3dconf/frontend/afterModelLoaded', function({ store }) {
  console.log('Model ready, active layers applied');
}, 10);
```

---

#### `wp3dconf/frontend/canvas/apply`

Fires when a layer change is applied to the 3D canvas. Use this to hook into the rendering pipeline for custom material or mesh logic.

| Parameter      | Type   | Description |
|----------------|--------|-------------|
| `uid`          | String | UID of the layer being applied |
| `layer`        | Object | The layer object |
| `targetMeshes` | Array  | Three.js mesh objects targeted by this layer |
| `customData`   | Object | Any custom data attached to the layer |
| `store`        | Object | The Alpine.js store instance |

```js
addAction('wp3dconf/frontend/canvas/apply', function({ uid, layer, targetMeshes, store }) {
  targetMeshes.forEach(mesh => {
    // Custom mesh manipulation
  });
}, 10);
```

---

## JavaScript Filters

#### `wp3dconf/frontend/price/layer_amounts`

Filters a single active layer's own `{ price, salePrice }` pair before it's added into the running total. Runs once per active layer, before `wp3dconf/frontend/price/layer` (which instead filters the accumulated running total after each layer is folded in). Use this one when you need to change what a specific layer itself is worth; use `price/layer` when you need to adjust the total as a whole.

| Parameter | Type   | Description |
|-----------|--------|-------------|
| `amounts` | Object | `{ price, salePrice }` for this layer, read from its settings |
| `context` | Object | `{ uid, layer, store }` |

```js
addFilter('wp3dconf/frontend/price/layer_amounts', function(amounts, { uid, layer, store }) {
  if (uid === 'my-layer-uid') {
    amounts.price += 5;
  }
  return amounts;
}, 10);
```

---

#### `wp3dconf/frontend/price/layer`

Filters the price contribution of an individual layer. Return a modified `total` to override the default price calculation.

| Parameter | Type   | Description |
|-----------|--------|-------------|
| `total`   | Number | The current price total for this layer |
| `uid`     | String | UID of the layer |
| `layer`   | Object | The layer object |
| `store`   | Object | The Alpine.js store instance |

```js
window.addEventListener('wp3dconf/frontend:ready', function() {
  const { addFilter } = window.WP3dConf;

  addFilter('wp3dconf/frontend/price/layer', function(total, { uid, layer, store }) {
    if (uid === 'my-layer-uid') {
      return total * 2;
    }
    return total;
  }, 10);
});
```

---

#### `wp3dconf/frontend/summary/layerCustomData`

Filters the custom data shown for a layer in the summary popup. Return `null` to omit the entry entirely (used by Custom Fields to hide a switch's raw `true` value, since the layer name already says everything).

| Parameter | Type          | Description |
|-----------|---------------|-------------|
| `data`    | Object\|null  | The layer's custom data (`store.customData[uid]`), or `null` |
| `context` | Object        | `{ uid, store }` |

```js
addFilter('wp3dconf/frontend/summary/layerCustomData', function(data, { uid, store }) {
  if (!data) return data;
  return { ...data, extra: 'note' };
}, 10);
```

---

#### `wp3dconf/frontend/submit/blocked`

Filters whether the add-to-cart / Get a Quote submission should be blocked. Return `true` to block. Used by Custom Fields to block submission while a required field is empty.

| Parameter | Type    | Description |
|-----------|---------|-------------|
| `blocked` | Boolean | Whether submission is currently blocked |
| `context` | Object  | `{ store, form, context }` — `form` is the form element, `context` is `'cart'` or `'quote'` |

```js
addFilter('wp3dconf/frontend/submit/blocked', function(blocked, { store, context }) {
  if (blocked) return blocked; // Don't override an existing block.
  return myValidationFails(store) ? true : blocked;
}, 10);
```

---

#### `wp3dconf/frontend/storeValuesToRetrieveParams`

Filters the request parameters sent when generating a shareable link (the `?key=` flow). Use this to append additional fields to the payload the server stores.

| Parameter | Type   | Description |
|-----------|--------|-------------|
| `params`  | Object | Request params (`action`, `nonce`, `active_layers`, `custom_data`) |
| `store`   | Object | The Alpine.js store instance |

```js
addFilter('wp3dconf/frontend/storeValuesToRetrieveParams', function(params, store) {
  params.my_field = store.myValue;
  return params;
}, 10);
```

---

## PHP Actions

#### `wp3dconf/frontend/after_enqueue`

Fires after all frontend scripts and styles have been enqueued.

```php
add_action( 'wp3dconf/frontend/after_enqueue', function() {
  wp_enqueue_script( 'my-frontend-addon', plugin_dir_url( __FILE__ ) . 'frontend-addon.js' );
} );
```

---

#### `wp3dconf/frontend/before_skin_display`

Fires before the configurator skin is rendered.

| Parameter        | Type   | Description |
|------------------|--------|-------------|
| `$skin_instance` | Object | The skin class instance |
| `$post_id`       | int    | The configurator post ID |

---

#### `wp3dconf/frontend/after_skin_display`

Fires after the configurator skin has been rendered.

| Parameter        | Type   | Description |
|------------------|--------|-------------|
| `$skin_instance` | Object | The skin class instance |
| `$post_id`       | int    | The configurator post ID |

---

#### `wp3dconf/frontend/form/hidden_fields`

Fires inside the configurator form after the default hidden fields. Use this to output additional hidden inputs.

| Parameter | Type   | Description |
|-----------|--------|-------------|
| `$skin`   | Object | The current skin instance |

The `$skin` object exposes a `$skin->store` property - an Alpine.js store reference string (e.g. `$store.wp3dconf_123`). Use this to bind hidden fields to live store values via `x-bind:value`.

```php
add_action( 'wp3dconf/frontend/form/hidden_fields', function( $skin ) {
  // Static value
  echo '<input type="hidden" name="my_field" value="my_value">';

  // Bind to Alpine.js store - value updates reactively as the customer configures
  echo '<input type="hidden" name="active_layers"
    x-bind:value=\'JSON.stringify(' . esc_attr( $skin->store ) . '.activeLayers)\'>';

  // Bind attachment ID
  echo '<input type="hidden" name="attachment_id"
    x-bind:value=\'' . esc_attr( $skin->store ) . '.attachmentID\'>';

  // Bind all custom layer data (text, range, etc.)
  echo '<input type="hidden" name="custom_data"
    x-bind:value=\'JSON.stringify(' . esc_attr( $skin->store ) . '.customData)\'>';
}, 10, 1 );
```

---

### Controls Rendering Actions

These actions fire around the rendering of each control element in the frontend HTML output.

| Hook | Parameters | Description |
|------|-----------|-------------|
| `wp3dconf/frontend/controls/before_html` | `$skin` | Before the entire controls wrapper |
| `wp3dconf/frontend/controls/after_html` | `$skin` | After the entire controls wrapper |
| `wp3dconf/frontend/controls/before_inner_html` | `$skin` | Before the inner controls container |
| `wp3dconf/frontend/controls/after_inner_html` | `$skin` | After the inner controls container |
| `wp3dconf/frontend/controls/before_group_html` | `$skin` | Before a group block is rendered |
| `wp3dconf/frontend/controls/after_group_html` | `$skin` | After a group block is rendered |
| `wp3dconf/frontend/controls/before_control_item` | `$layer, $skin` | Before a control item wrapper |
| `wp3dconf/frontend/controls/before_control_item_inner` | `$layer, $skin` | Before a control item inner container |
| `wp3dconf/frontend/controls/before_layer_html` | `$layer, $skin` | Immediately before the layer-type-specific markup (i.e. before `layer_{type}_html` fires) |
| `wp3dconf/frontend/controls/layer_{type}_html` | `$layer, $skin` | Inside a control item for a specific layer type. Replace `{type}` with the layer type slug (e.g. `layer_color_html`) |
| `wp3dconf/frontend/controls/after_layer_html` | `$layer, $skin` | Immediately after the layer-type-specific markup |
| `wp3dconf/frontend/controls/after_control_item_inner` | `$layer, $skin` | After a control item inner container |
| `wp3dconf/frontend/controls/before_control_icon` | `$uid, $skin` | Before the control icon element |
| `wp3dconf/frontend/controls/after_control_icon` | `$uid, $skin` | After the control icon element |

```php
// Add content after a control icon
add_action( 'wp3dconf/frontend/controls/after_control_icon', function( $uid, $skin ) {
  echo '<span class="my-badge">New</span>';
}, 10, 2 );

// Custom HTML for a specific layer type
add_action( 'wp3dconf/frontend/controls/layer_color_html', function( $layer, $skin ) {
  echo '<span class="color-swatch" style="background:' . esc_attr( $layer['value'] ) . '"></span>';
}, 10, 2 );
```

### Controls Rendering Filters

These filter the classes and attributes rendered on each control item wrapper.

| Hook | Parameters | Description |
|------|-----------|-------------|
| `wp3dconf/frontend/controls/wrapper_classes` | `$classes` | Filters the classes on the outermost controls wrapper element |
| `wp3dconf/frontend/controls/item_classes` | `$classes` | Filters the classes array (`selector`, `class`, `control_type`, `layer_type`) applied to a single control item |
| `wp3dconf/frontend/controls/item_attributes` | `$attr, $uid, $skin` | Filters the full HTML-attribute-string array (`class`, `uid`, Alpine bindings) rendered on a control item |

```php
add_filter( 'wp3dconf/frontend/controls/item_classes', function( $classes ) {
  $classes['my-addon'] = 'my-addon-class';
  return $classes;
} );
```

---

## Skins

#### `wp3dconf/register_skins`

Fires on `plugins_loaded` (priority 20). Use this hook to register a custom skin class via `Skin_Registry::register()`.

| Parameter    | Type           | Description |
|--------------|----------------|-------------|
| `$skin_name` | String         | Unique skin identifier |
| `$skin`      | String\|Object | Fully-qualified class name string or an object instance |

```php
use WP3DCONF\Skin_Registry;

add_action( 'wp3dconf/register_skins', function() {
  Skin_Registry::register( 'my-skin', My_Custom_Skin::class );
} );
```

You can also pass an object instance. The skin system will `clone` it and call `set_post_id()` automatically.

```php
use WP3DCONF\Skin_Registry;

add_action( 'wp3dconf/register_skins', function() {
  Skin_Registry::register( 'my-skin', new My_Custom_Skin() );
} );
```

Your skin class must extend `WP3DCONF\Skin_Base` and implement the `display()` method.

```php
namespace My_Addon;

use WP3DCONF\Skin_Base;
use WP3DCONF\Skin_Registry;

class My_Custom_Skin extends Skin_Base {

  protected $skin_id = 'my-skin';

  public function display( $output = true ) {
    // Render your skin HTML
  }
}
```

---

#### `wp3dconf/registered_skins`

Filters the full array of registered skins after `wp3dconf/register_skins` has fired. Use this to modify or remove existing skin registrations.

| Parameter | Type  | Description |
|-----------|-------|-------------|
| `$skins`  | Array | All registered skins keyed by skin identifier |

```php
add_filter( 'wp3dconf/registered_skins', function( $skins ) {
  unset( $skins['accordion'] );
  return $skins;
} );
```