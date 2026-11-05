# Range - Developer Guide

**Audience:** developers extending or integrating the Range layer: themes and add-on plugins.
**Not for:** shop owners (see the Range user guide on docs-3d.wpconfigurator.com).
**Applies to:** plugin v1.3.5
**Last updated:** 2026-10-29

Read [extend-modules.md](extend-modules.md) first for the general module contract, and
[custom-fields.md](custom-fields.md) for the module the Range belongs to. This guide covers only what is
specific to the Range layer type and where you can hook into it.

> **Pro only.** The Range is a layer type of the **Custom Fields** module (`custom-fields`), which is not in
> the Lite package and is off by default. Guard your integration with
> `Modules::is_enabled( 'custom-fields' )` in PHP and `window.WP3dConf.customFields` in JS.

---

## 1. What it does

A **Range layer** holds one number on a scale (Minimum, Maximum, Step): a width, a length, a quantity. The
shopper sets it with a slider, a number box or both. It is priced as a flat price plus either a matching
**price tier** or a **price per unit**.

Every value is **normalised**: snapped onto the scale and clamped to it. The browser normalises and prices
it while the shopper builds; the server normalises and prices it again on every submission and never
trusts the browser's figures.

```
slider / box --> handleRangeInput / handleRangeChange --> normalise --> store.customData[uid] = { value }
                                                                         |
                                     wp3dconf/frontend/price/layer_amounts --> live total
                                                                         |
     submit --> server re-normalises the posted value --> wp3dconf/data/custom_data_layer_price
                                                                         |
                                       cart / checkout / quote / PDF:  "Width (1200 mm) ($51.00)"
```

A Range is **always selected**: it's activated with its default on page load and never deactivates itself,
so it always appears in the summary and the cart. It has no Required setting.

---

## 2. File map

```
modules/custom-fields/
├── class-custom-fields.php   # Module_Base subclass: layer types, settings, payloads, pricing + summary filters
├── class-fields.php          # Fields: type table, settings schema, and the Range helpers (normalise, price, tiers)
└── assets/frontend/
    ├── custom-fields.js      # hand-authored runtime, no build step
    └── custom-fields.css     # .wp3dconf-range* rules
templates/layer/
├── layer-field.php           # shared wrapper: label, description, live price
└── layer-range.php           # the slider and/or number box
```

**There is no Vite build for the frontend runtime.** `custom-fields.js` is served as written;
`Module_Base` falls back to `<slug>.js` when no `<slug>.min.js` exists. The editor controls (the Range
preview and the Price Tiers repeater) are part of the core editor bundle, not of the module.

---

## 3. Data model

### 3.1 Layer settings (saved with the configurator)

Stored in `_wp3dconf_layer_settings` under the layer's `uid`. Frontend reads them at
`window.WP3dConf.layers[uid].settings`.

| Key | Type | Default | Notes |
|---|---|---|---|
| `range_min` / `range_max` | number | `0` / `100` | Non-numeric reads as the default. Swapped when Maximum is below Minimum. |
| `range_step` | number | `1` | `<= 0` or non-numeric reads as 1. Decimals allowed. |
| `field_default` | number | the minimum | Snapped onto the scale like any other value |
| `range_display` | string | `both` | `both`, `input` (box only) or `slider` (slider only) |
| `range_suffix` | string | `''` | The unit |
| `price` / `sale_price` | number | `0` | Flat price, always charged. The sale price counts only when `0 < sale < regular`. |
| `range_unit_price` | number | `0` | Charged per unit when no tier matches |
| `range_price_tiers` | array | `[]` | Rows of `{ from, to, price, per_unit }`. An empty `from` / `to` is `null`. |
| `label`, `field_description` | string | layer name / `''` | Field Settings › Label and Description |

```json
{
  "Kx3f9a": {
    "label": "Width",
    "range_min": 600, "range_max": 4800, "range_step": 10, "field_default": 1200,
    "range_display": "both", "range_suffix": "mm",
    "price": 15, "range_unit_price": 0.02,
    "range_price_tiers": [
      { "from": null, "to": 1000, "price": 20, "per_unit": false },
      { "from": 1010, "to": 2000, "price": 0.03, "per_unit": true }
    ]
  }
}
```

Saved settings can be out of order or incomplete (a hand-edited or imported configurator). **Never read
`range_min` and friends raw**; ask the `Fields` helpers in section 5 for the effective scale, value and
price.

### 3.2 Custom data (per shopper selection)

`store.customData[uid]` on the frontend, `custom_data` in cart/quote payloads:

```json
{ "value": 1200 }
```

- `value` is `Fields::VALUE_KEY`, the same key every field type uses.
- The stored value is what the browser sent. **Read it through `Fields::normalize_range_value()`** to get
  what was charged: 999 posted for a 0–100 scale is charged, printed and read by formulas as 100.

### 3.3 Frontend globals

| Global | Content |
|---|---|
| `window.WP3dConf.customFields.numericTypes` | `['range']` — test a layer with this rather than comparing to `'range'` |
| `window.WP3dConf.customFields.valueLabels` | Summary label per type (`range`: "Value") |
| `window.WP3dConf.layers[uid]` | Layer, including `type` and `settings` |

---

## 4. Extension points

### 4.1 How the Range plugs into `Module_Base`

The Range has no class of its own. `Custom_Fields` registers it alongside the other field types, through
the standard overrides:

| Method | What it does for the Range |
|---|---|
| `get_slug()` | `'custom-fields'` |
| `layer_types()` | Registers every field type, `range` included, from `Fields::get_types()`. The Range is declared with the `numeric` capability flag. |
| `layer_settings_controls( $settings )` | Adds the Range's Field Settings sections, gated to numeric types, and Price Per Unit / Price Tiers in Price Settings |
| `frontend_payload( $post_id )` | Becomes `window.WP3dConf.customFields` (section 3.3) |
| `editor_payload( $post_id )` | Becomes `window.editorData.customFields`, with the preview and tier strings |
| `init()` | Hooks the render action, the pricing and summary filters, and the kses allowance |

**Adding a similar layer type.** `Fields::get_types()` has no filter, so an add-on can't add a field type
to Custom Fields. Build your own module with `layer_type()` instead, following
[extend-modules.md](extend-modules.md). To give it the same scale and pricing as a Range, call the public
`Fields` helpers (section 5) from your own pricing filters rather than reimplementing them.

### 4.2 PHP hooks this module uses (and you can also use)

| Hook | Type | Args | Purpose here |
|---|---|---|---|
| `wp3dconf/frontend/controls/layer_range_html` | action | `$layer`, `$skin` | `Fields::render()` prints the control through the two templates |
| `wp3dconf/data/custom_data_layer_price` | filter | `$price`, `$layer`, `$field_data`, `$data` | `field_layer_price()` returns the Range's `price` / `sale_price` pair |
| `wp3dconf/data/custom_data_string` | filter | `$string`, `$uid`, `$custom_data`, `$data` | `field_custom_data_string()` returns `1200 mm` |
| `wp3dconf/utils/allowed_html_tags` | filter | `$tags` | Allows `min`, `max`, `step`, `inputmode` and the `aria-*` attributes on `input` |

The host also runs `wp3dconf/data/manipulate_layer_price` (`$price`, `$layer`, `$data`, `$custom_data`,
`$summary_uids`) after every layer's price is worked out. It is the last word on a layer's price wherever
the server builds a summary: cart, checkout, orders, Get a Quote, Contact Form 7 and the PDF.

> `custom_data_layer_price` passes **this layer's** custom data third and the `Data_Repository` fourth.

**Adjusting a Range's price** from your own plugin: filter `wp3dconf/data/manipulate_layer_price` at a
priority above 10, and make the same change in JS (section 4.3) — the shopper sees the browser's total
but is charged the server's.

```php
use WP3DCONF\CustomFields\Fields;
use WP3DCONF\Utils;

// A 10 % surcharge on any Range over 2000.
add_filter( 'wp3dconf/data/manipulate_layer_price', function ( $price, $layer, $data, $custom_data ) {
	if ( ! Fields::is_numeric_type( $layer['type'] ?? '' ) ) {
		return $price;
	}

	$settings = $layer['settings'] ?? array();
	$value    = Fields::normalize_range_value( $settings, $custom_data[ $layer['uid'] ]['value'] ?? null );

	if ( $value > 2000 ) {
		$places         = Utils::get_price_decimals(); // 90 × 1.1 is 99.00000000000001 otherwise
		$price['price'] = round( (float) $price['price'] * 1.1, $places );
		if ( (float) $price['sale_price'] > 0 ) {
			$price['sale_price'] = round( (float) $price['sale_price'] * 1.1, $places );
		}
	}

	return $price;
}, 20, 4 );
```

**Changing the summary text** (cart, emails, PDF): filter `wp3dconf/data/custom_data_string` at a priority
above 10, and return `$string` untouched for other layers. The result is escaped when printed. To add
markup after a row, use `wp3dconf/data/custom_data_html` and escape everything you add: that HTML reaches
email bodies and the PDF.

```php
use WP3DCONF\CustomFields\Fields;

add_filter( 'wp3dconf/data/custom_data_string', function ( $string, $uid, $custom_data, $data ) {
	if ( ! Fields::is_numeric_type( $data->get_layer_type( $uid ) ) ) {
		return $string;
	}

	return $string . ' (internal width)';
}, 20, 4 );
```

**Reading what a cart item was charged for:**

```php
use WP3DCONF\Configurator_Utils;
use WP3DCONF\CustomFields\Fields;
use WP3DCONF\Data_Repository;

$data        = new Data_Repository( (int) $cart_item['config_id'] );
$custom_data = Configurator_Utils::decode_custom_data( $cart_item['custom_data'] );

foreach ( (array) $data->get_layers() as $uid => $layer ) {
	if ( ! Fields::is_numeric_type( $layer['type'] ?? '' ) ) {
		continue;
	}

	$settings = $layer['settings'] ?? array();
	$value    = Fields::normalize_range_value( $settings, $custom_data[ $uid ]['value'] ?? null );
	$amounts  = Fields::range_amounts( $settings, $value ); // array( 'price' => …, 'sale_price' => … )
}
```

### 4.3 JS hooks this module uses (and you can also use)

Register inside the `wp3dconf/frontend:ready` event (see [hooks-frontend.md](hooks-frontend.md)).

| Hook | Type | Payload | Purpose here |
|---|---|---|---|
| `wp3dconf/frontend/storeInitialized` | action | `{ store }` | Adds the store methods in section 4.4 |
| `wp3dconf/frontend/data/initialized` | action | `{ store }` | Activates every Range with its normalised default, or corrects a value restored from a shareable link |
| `wp3dconf/frontend/price/layer_amounts` | filter | `amounts`, `{ uid, layer, store }` | Prices the Range; returns `{ price, salePrice }` |
| `wp3dconf/frontend/summary/layerCustomData` | filter | `data`, `{ uid, store }` | Relabels the summary popup line as `{ Value: "1200 mm" }` |

The runtime **fires** `wp3dconf/frontend/layer/customDataUpdated` (`{ uid, layer, store }`) on every change
after the first, and `wp3dconf/frontend/layer/activated` on the first. `wp3dconf/frontend/data/ready`
(`{ store }`) fires once defaults are in and the first total is worked out — a safe point to read every
Range. A Range never goes through the option-click path, so it never fires
`wp3dconf/frontend/user/option_clicked` and never triggers a [Model Animation](model-animation.md).

All filters must return their first argument untouched for layers that are not Ranges. The mirror of the
PHP surcharge above, registered after the module's own filter:

```js
window.addEventListener('wp3dconf/frontend:ready', () => {
  const { addFilter } = window.WP3dConf;
  const { numericTypes } = window.WP3dConf.customFields || { numericTypes: [] };

  addFilter('wp3dconf/frontend/price/layer_amounts', (amounts, { uid, layer, store }) => {
    if (!layer || !numericTypes.includes(layer.type) || store.normalizeFieldValue(uid) <= 2000) {
      return amounts;
    }

    return {
      price: amounts.price * 1.1,
      salePrice: amounts.salePrice > 0 ? amounts.salePrice * 1.1 : amounts.salePrice,
    };
  }, 20);
});
```

A [Formula](formula.md) that reads the Range's *price* picks the adjustment up on both sides.

### 4.4 Driving the layer from your own code

The runtime adds these to the frontend store (passed in every hook payload):

| Call | Effect |
|---|---|
| `store.normalizeFieldValue(uid, raw?)` | The normalised number; omit `raw` to normalise the stored value |
| `store.handleRangeChange(uid, value)` | Sets the value as a shopper would: snaps it, syncs the slider and box, reprices the page |
| `store.handleRangeInput(uid, value)` | The same, debounced by 120 ms per uid (what the slider calls while dragging) |
| `store.getFieldPrice(uid)` | The effective price as plain text, or `''` at 0 |
| `store.getLayerAmounts(uid)` | `{ price, salePrice }` for the current pricing pass |

Example: react to a change, and set a value from your own button:

```js
window.addEventListener('wp3dconf/frontend:ready', () => {
  const { addAction } = window.WP3dConf;
  const { numericTypes } = window.WP3dConf.customFields || { numericTypes: [] };

  addAction('wp3dconf/frontend/layer/customDataUpdated', ({ uid, layer, store }) => {
    if (layer && numericTypes.includes(layer.type)) {
      document.dispatchEvent(new CustomEvent('my-addon:width', { detail: { uid, value: store.normalizeFieldValue(uid) } }));
    }
  });

  addAction('wp3dconf/frontend/data/ready', ({ store }) => {
    document.querySelector('#my-preset-large')?.addEventListener('click', () => {
      store.handleRangeChange('YOUR_LAYER_UID', 2400);
    });
  });
});
```

> Don't write `store.customData[uid].value` directly: it skips the snapping, the control sync and the
> pricing pass.

---

## 5. Normalisation and pricing

Every price, summary line and formula reads a Range's value through one rule:

1. Read the scale: a non-numeric Minimum is 0, Maximum 100, Step 1; a Step of 0 or less is 1; Minimum
   and Maximum are swapped if reversed.
2. An empty or non-numeric value becomes the default, or the minimum.
3. Snap to the nearest step **counted from the minimum**, round to the precision of the step and minimum,
   and clamp to the scale.

So on 5–100 in steps of 10 the values are 5, 15 … 95, and on 0–100 in steps of 3 the slider stops at 99.

The price is:

```
price      = flat price + charge
sale_price = sale price + charge          (only when a sale price is set)

charge     = first tier with from ≤ value ≤ to:  its price, or value × price when per_unit
             no tier matches:                   value × Price Per Unit
```

Only the first 50 tier rows count. The total then counts the **effective** price: the sale price when
`0 < sale < regular`, otherwise the regular price.

### Public helpers (`WP3DCONF\CustomFields\Fields`)

| Helper | Returns |
|---|---|
| `Fields::is_numeric_type( $type )` | Whether a layer type is numeric (a Range) |
| `Fields::normalize_range_value( $settings, $raw )` | The value as charged |
| `Fields::range_amounts( $settings, $value )` | The Range's own `price` / `sale_price` pair for a normalised value |
| `Fields::price_tier( $settings, $value )` | The matching tier `{ from, to, price, per_unit }`, or `null` |

Use these helpers rather than your own copy of the maths.

---

## 6. Customising without editing the module

### 6.1 Overriding templates from a theme

Both templates load through `wp3dconf_layer_template()` and the plugin's template loader, so they can be
overridden under the active theme:

```
{theme}/wp3dconf/wp-3d-configurator/layer/layer-range.php
{theme}/wp3dconf/wp-3d-configurator/layer/layer-field.php   (the frame around every field type)
```

The template receives `$args['field']` with the values already resolved: `min`, `max`, `step`, `value`,
`suffix`, `display`, `label`, `field_id`, and the Alpine expressions `range_input` and `range_change`.
Abridged output, with *Slider and box* chosen:

```html
<div class="wp3dconf-range wp3dconf-range-both" x-data="{ v: 1200 }">
  <input type="range" class="wp3dconf-field-input wp3dconf-range-slider" data-uid="Kx3f9a"
         min="600" max="4800" step="10" x-bind:value="v"
         x-on:input="v = $event.target.value; <range_input>" x-on:change="<range_change>" />
  <span class="wp3dconf-range-readout">
    <input type="number" class="wp3dconf-field-input wp3dconf-range-box" data-uid="Kx3f9a"
           min="600" max="4800" step="10" inputmode="decimal" x-bind:value="v" x-on:change="<range_change>" />
    <span class="wp3dconf-range-suffix">mm</span>
  </span>
</div>
```

Rules an override must respect:

- **Keep the `.wp3dconf-range` wrapper with `x-data="{ v: … }"`, and `data-uid` on each input.** The
  runtime finds the control with `.wp3dconf-range [data-uid="<uid>"]` and writes the snapped value back
  into `v`.
- Bind `range_input` on `x-on:input` and `range_change` on `x-on:change`. Don't bind the store with
  `x-model`.
- The output passes through `wp_kses()`. Use `x-text`, never `x-html`, and only the attributes the module
  allows on `input`. **Anything else is stripped silently.** Widen with `wp3dconf/utils/allowed_html_tags`,
  and never allow event-handler attributes or Alpine directives beyond what the core list permits.
- Escape every echoed value at the point of output.

### 6.2 Styling

`assets/frontend/custom-fields.css`. Override from your theme with more specific selectors:

| Selector | Element |
|---|---|
| `.wp3dconf-control-item.wp3dconf-layer-type-range` | The list item (`wp3dconf-layer-hidden` / `-disabled` added by Conditional Logic) |
| `.wp3dconf-field.wp3dconf-field-range` | The field wrapper |
| `.wp3dconf-range` + `-both` / `-input` / `-slider` | The control, by input style |
| `input.wp3dconf-range-slider`, `input.wp3dconf-range-box` | The slider and the number box |
| `.wp3dconf-range-value`, `.wp3dconf-range-suffix` | The value text (slider only) and the unit |

With *Slider only*, the wrapper's modifier class is `wp3dconf-range-slider`, the same class as the slider
input. Target the input as `input.wp3dconf-range-slider`.

---

## 7. Debugging

| Symptom | Check |
|---|---|
| Cart price differs from the live price | A price change made on one side only. Mirror it in `manipulate_layer_price` and `price/layer_amounts`. |
| My `layer_amounts` change is overwritten | It ran before the module's own filter. Register at a priority above 10. |
| The slider stops short of the maximum | `max − min` isn't a whole number of steps; steps count from the minimum |
| Range renders but doesn't respond | The configurator has no 3D model, so no frontend scripts load; or an override dropped the `.wp3dconf-range` wrapper or `data-uid` |
| A custom attribute disappears from an override | kses stripped it; allow it through `wp3dconf/utils/allowed_html_tags` |
| A Range renders nothing at all | Custom Fields is switched off, or the site runs Lite |
