# Formula - Developer Guide

**Audience:** developers extending or integrating the Formula layer: themes and add-on plugins.
**Not for:** shop owners (see the Formula user guide on docs-3d.wpconfigurator.com).
**Applies to:** plugin v1.3.5
**Last updated:** 2026-10-29

Read [extend-modules.md](extend-modules.md) first for the general module contract. This guide covers only
what is specific to this module and where you can hook into it.

> **Pro only.** The Formula module (`formula`) is not in the Lite package and is off by default. Guard
> your integration with `Modules::is_enabled( 'formula' )` in PHP and `window.WP3dConf.formula` in JS.

---

## 1. What it does

A **Formula layer** is a computed layer type in the Model Structure tab. It evaluates an expression over
other layers — `{width} * {drop} / 1000000 * 45` — and either **adds the result to the total** as its own
price (price mode), or **only shows** it, such as an area, a weight or a count (value mode). Variables read
another layer's *value* (a Range's number, how many options are chosen) or its *price*, and a formula can
read other formulas.

The expression is evaluated twice: in the browser, so the total updates while the shopper builds, and
again on the server, which **recomputes every result** from the posted selection and never trusts the
browser's.

```
any layer changes --> pricing pass --> wp3dconf/frontend/price/layer_amounts
                                         |  Resolver reads each variable, Expression evaluates
                                         v
                           store.customData[uid] = { result }    (display only)
                           price mode: { price: max(0, round(result)), salePrice: 0 }
                                         |
     submit --> wp3dconf/data/summary_uids (adds every priced formula back in)
            --> wp3dconf/data/manipulate_layer_price (recomputes on the server)
            --> cart / quote / PDF:  "Glass ($81.00)"  or  "Opening area (1.80 m²)"
```

Every formula is **always selected**: it's activated on page load and re-activated if anything deselects
it. A price-mode formula left out of a request is added back on the server, so a charge can't be dodged by
editing the request.

---

## 2. File map

```
modules/formula/
├── class-formula.php      # Module_Base subclass: layer type, settings, payloads, pricing + summary + sanitising filters
├── class-expression.php   # Expression: tokenizer, parser, evaluator, sanitize(), round_price() — no WordPress dependency
├── class-resolver.php     # Resolver: what number each layer contributes, chaining, cycles
└── assets/frontend/
    ├── formula.min.js     # built frontend runtime (IIFE)
    └── formula.css        # hand-written
common/_dev/modules/formula/frontend/
├── main.ts                # runtime source: when formulas are evaluated, where results go
└── resolver.ts            # browser twin of class-resolver.php
templates/layer/layer-formula.php   # the read-out row
```

The frontend runtime is built from `common/_dev/modules/formula/frontend/` by the standard module build
(see [extend-modules.md](extend-modules.md)). The editor control (variables, expression, live preview) is
part of the core editor bundle, not of the module.

---

## 3. Data model

### 3.1 Layer settings (saved with the configurator)

Stored in `_wp3dconf_layer_settings` under the layer's `uid`. Frontend reads them at
`window.WP3dConf.layers[uid].settings`.

| Key | Type | Default | Notes |
|---|---|---|---|
| `formula` | object | `{ expression: '', variables: [] }` | One key, so the expression and its variables are never saved apart |
| `formula.expression` | string | `''` | Up to 500 characters |
| `formula.variables` | array | `[]` | `{ name, uid, source }`: `name` matches `^[A-Za-z_][A-Za-z0-9_]*$` and is unique; `source` is `value` or `price` |
| `formula_result_type` | string | `price` | `price` (*Add to the price*) or `value` (*Show only*). Anything but `value` means price mode. |
| `formula_show` | bool | `true` | Whether the row is printed on the frontend |
| `formula_prefix` / `formula_suffix` | string | `''` | Value mode only |
| `formula_decimals` | number | `2` | Value mode only, 0–10. A charged result follows the store's price decimals. |

```json
{
  "Fg7a2c": {
    "formula": {
      "expression": "{width} * {height} / 1000000 * 45",
      "variables": [
        { "name": "width",  "uid": "Kx3f9a", "source": "value" },
        { "name": "height", "uid": "Lp2q8d", "source": "value" }
      ]
    },
    "formula_result_type": "price",
    "formula_show": true
  }
}
```

A formula has no `price` / `sale_price` of its own: its price is its result. Variable rows with a bad or
repeated name, or an empty uid, are ignored when the formula is read, so always read the setting through
`Formula::parse_setting( $settings['formula'] )`.

### 3.2 Custom data (per shopper selection)

`store.customData[uid]` on the frontend, `custom_data` in cart/quote payloads:

```json
{ "result": 81 }
```

- `result` (`Formula::RESULT_KEY`) is the browser's unrounded result, **display only**. The server casts
  the posted value to a float and never prices or prints it.
- To get a trustworthy result on the server, recompute it with `Resolver` (section 4.2).

### 3.3 Frontend globals

| Global | Content |
|---|---|
| `window.WP3dConf.formula` | `{ layerType: 'formula', valueKey: 'value', resultKey: 'result', i18n: { resultLabel } }` (from `frontend_payload()`) |
| `window.WP3dConf.layers[uid]` | Layer, including `type` and `settings` |

---

## 4. Extension points

### 4.1 Adding a similar layer type

The Formula module is a `Module_Base` subclass using the standard overrides:

| Method | What this module returns |
|---|---|
| `get_slug()` | `'formula'` |
| `layer_type()` | `array( 'slug' => 'formula', 'name' => 'Formula', 'icon' => 'code_blocks' )` for the Model Structure list |
| `layer_settings_controls( $settings )` | Adds the Formula Settings group, gated by `layer_type === 'formula'`, and hides Price, Sale Price, Color and Icon on formula layers |
| `editor_payload( $post_id )` | Becomes `window.editorData.formula` |
| `frontend_payload( $post_id )` | Becomes `window.WP3dConf.formula` |
| `init()` | Hooks the render action and the data and sanitising filters in section 4.2 |

A layer type that **prices itself from other layers** — rather than from a shopper's choice — needs the
same three server-side hooks this module uses: `wp3dconf/data/summary_uids` so it's always in the summary,
`wp3dconf/data/active_layers` so its row is printed, and `wp3dconf/data/manipulate_layer_price` to price
it.

### 4.2 PHP hooks this module uses (and you can also use)

| Hook | Type | Args | Purpose here |
|---|---|---|---|
| `wp3dconf/frontend/controls/layer_formula_html` | action | `$layer`, `$skin` | Renders the row via `wp3dconf_layer_template()`; nothing when `formula_show` is off |
| `wp3dconf/data/summary_uids` | filter | `$uids`, `$data`, `$custom_data` | Adds every price-mode formula, and its parent, that a request left out |
| `wp3dconf/data/active_layers` | filter | `$active_layers`, `$data` | Puts price-mode formulas back into the stored active-layer map, so the printed rows explain the charge |
| `wp3dconf/data/manipulate_layer_price` | filter | `$price`, `$layer`, `$data`, `$custom_data`, `$summary_uids` | Price mode: `max( 0, result )` rounded to the store's decimals. Value mode: 0. |
| `wp3dconf/data/custom_data_string` | filter | `$string`, `$uid`, `$custom_data`, `$data`, `$summary_uids` | Price mode: `''` (the line's price already shows it). Value mode: `1.80 m²`. |
| `wp3dconf/data/sanitize_custom_data_value` | filter | `$clean`, `$value`, `$key`, `$uid` | Casts a posted `result` to a float |
| `wp3dconf/data/sanitize_layer_setting` | filter | `$clean`, `$value`, `$key` | Sanitises `formula.expression` on an editor save |
| `sanitize_post_meta__wp3dconf_layer_settings` | filter (WordPress) | `$meta_value` | Sanitises every expression on any other write: import, duplicate, `update_post_meta()` |

All `wp3dconf/data/*` filters fire for every layer: return the incoming value untouched for layers that
aren't yours.

**Adjusting a formula's charge** from your own plugin: filter `wp3dconf/data/manipulate_layer_price` at a
priority above 10, and make the same change in JS (section 4.3) — the shopper sees the browser's total but
is charged the server's.

```php
use WP3DCONF\Formula\Formula;

// A minimum charge of 25 for every priced formula.
add_filter( 'wp3dconf/data/manipulate_layer_price', function ( $price, $layer ) {
	if ( ! Formula::is_formula( $layer ) || ! Formula::is_price_mode( $layer['settings'] ?? array() ) ) {
		return $price;
	}

	$price['price'] = max( 25.0, (float) $price['price'] );

	return $price;
}, 20, 2 );
```

An adjusted charge doesn't reach other formulas: a formula reading another formula's *price* gets its
unrounded result.

**Reading a formula's result for a cart item.** Recompute it; never read the posted `result`:

```php
use WP3DCONF\Configurator_Utils;
use WP3DCONF\Data_Repository;
use WP3DCONF\Formula\Formula;
use WP3DCONF\Formula\Resolver;

$data        = new Data_Repository( (int) $cart_item['config_id'] );
$custom_data = Configurator_Utils::decode_custom_data( $cart_item['custom_data'] );
$active      = Configurator_Utils::flatten_all_uids( json_decode( stripslashes( $cart_item['active_layers'] ), true ) );

$result   = ( new Resolver( $data, $custom_data, $active ) )->result( 'Fg7a2c' );
$settings = ( (array) $data->get_layers() )['Fg7a2c']['settings'] ?? array();
$text     = Formula::format_result( $result, $settings ); // "1.80 m²"
```

A cart item's `active_layers` already include every price-mode formula. For a selection from anywhere
else, pass the uids through `wp3dconf/data/summary_uids` first, as the plugin does.

Other public helpers: `Formula::is_formula( $layer )`, `Formula::is_price_mode( $settings )`,
`Formula::shows_result( $settings )`, `Formula::parse_setting( $setting )`.

### 4.3 JS hooks this module uses (and you can also use)

Register inside the `wp3dconf/frontend:ready` event (see [hooks-frontend.md](hooks-frontend.md)).

| Hook | Type | Payload | Purpose here |
|---|---|---|---|
| `wp3dconf/frontend/storeInitialized` | action | `{ store }` | Adds `getFormulaResult()` and `getFormulaPrice()` to the store |
| `wp3dconf/frontend/data/initialized` | action | `{ store }` | Activates every formula with `{ result: 0 }` |
| `wp3dconf/frontend/price/layer_amounts` | filter | `amounts`, `{ uid, layer, store }` | Evaluates the formula, writes `customData[uid].result`, returns its charge as `{ price, salePrice: 0 }` (0 in value mode) |
| `wp3dconf/frontend/data/priceUpdated` | action | `{ store, regularPrice, total }` | Releases the pass's resolver; every result is current here |
| `wp3dconf/frontend/layer/deactivated` | action | `{ uid, layer, store }` | Re-activates a formula straight away |
| `wp3dconf/frontend/summary/layerCustomData` | filter | `data`, `{ uid, store }` | `{ Result: "1.80 m²" }` for a shown value-mode formula, `null` otherwise |

All filters must return their first argument untouched for layers that are not formulas. The mirror of
the PHP minimum charge above, registered after the module's own filter:

```js
window.addEventListener('wp3dconf/frontend:ready', () => {
  window.WP3dConf.addFilter('wp3dconf/frontend/price/layer_amounts', (amounts, { layer }) => {
    const priced = layer && 'formula' === layer.type && 'value' !== layer.settings?.formula_result_type;

    return priced ? { ...amounts, price: Math.max(25, amounts.price) } : amounts;
  }, 20);
});
```

### 4.4 Driving the layer from your own code

**Reading results.** Within a pricing pass, a layer's price is computed on demand, so a formula reading a
Range's price always gets this pass's figure. Read results once the pass is done:

| Call | Effect |
|---|---|
| `store.getFormulaResult(uid)` | The value-mode text — prefix, number and suffix — in the store's number format, e.g. `1.80 m²` |
| `store.getFormulaPrice(uid)` | The charge as plain money text, or `''` when it's 0 |
| `store.customData[uid].result` | The unrounded result |
| `store.getLayerAmounts(uid).price` | The rounded charge (0 in value mode) |

```js
window.addEventListener('wp3dconf/frontend:ready', () => {
  window.WP3dConf.addAction('wp3dconf/frontend/data/priceUpdated', ({ store }) => {
    const shown = store.getFormulaResult('YOUR_LAYER_UID');   // "1.80 m²"
  });
});
```

> Don't write `customData[uid].result` yourself: the next pass overwrites it, and the server ignores it.

**Making your own layer type readable by formulas.** A layer type from your module works in formulas with
nothing formula-specific on your side, as long as it follows the host's conventions:

1. **Store a value** at `customData[uid].value`: a number, a numeric string, a list (read as a count) or a
   boolean (read as 1 or 0). An active layer with no `value` reads 1.
2. **Activate the layer** while it's part of the configuration: `store.activateLayer(uid, { value })`. An
   inactive layer reads 0.
3. **Price it on both sides**: `wp3dconf/frontend/price/layer_amounts` in the browser, and
   `wp3dconf/data/custom_data_layer_price` or `wp3dconf/data/manipulate_layer_price` on the server. A
   formula's *price* source then reads the same figure on both sides.

If your value needs correcting before use (clamping, snapping), store the corrected value: formulas read
what's stored.

---

## 5. The expression language

```
{name}                      a variable
+ - * / % ^                 arithmetic (^ is power, right-associative)
< <= > >= = !=              one comparison per step — join more with and / or
and  or                     logic (lower case)
( )                         grouping
min max clamp round ceil floor abs sqrt if not
```

| Function | Arguments | Result |
|---|---|---|
| `min`, `max` | 1 or more | Smallest / largest |
| `clamp` | `x, low, high` | `x` held between `low` and `high` |
| `round` | `x, places = 0` | Half away from zero |
| `ceil`, `floor`, `abs` | 1 | As named |
| `sqrt` | 1 | 0 for a negative argument |
| `if` | `condition, then, else` | `then` when the condition isn't 0 |
| `not` | 1 | 1 when the argument is 0, else 0 |

Comparisons and logic give 1 or 0, so `{price} * (1 - 0.1 * ({qty} >= 10))` takes 10 % off from 10 up.
Division or remainder by zero gives 0, an unknown variable reads 0, and an expression that doesn't parse
evaluates to 0: a formula never throws. Limits: 500 characters and 10 levels of nesting.

**The function list is fixed**; an add-on can't register functions. To feed a formula something it can't
compute, expose it as a layer's value (section 4.4).

### 5.1 What a variable reads

| Layer type | `value` source | `price` source |
|---|---|---|
| `formula` | Its result | Its unrounded result in price mode, else 0 |
| `group`, `subgroup` | Number of direct children chosen | Sum of its chosen items' prices, subgroups included |
| `range` | Its normalised number (see [range.md](range.md)) | Its effective price, tiers included |
| `checkbox` | Number of entries ticked | Its effective price |
| `switch` | 1 when on, else 0 | Its effective price |
| `option`, `colorpicker`, any layer with no stored value | 1 | Its effective price |
| `textfield`, `textarea`, `select`, `radio` | The stored value's leading number (`"12 cm"` reads 12) | Its effective price |

Unchosen layers read 0, except a group's value and price and a formula's value. Deleted layers read 0. A
layer's *price* is its effective price (the sale price when `0 < sale < regular`) after every price filter,
yours included. If formulas read each other in a loop, every formula in the loop resolves to 0.

### 5.2 Validating, generating and saving expressions

If your plugin writes formulas — an importer, a template library, a migration — validate them with the same
engine (`WP3DCONF\Formula\Expression`, no WordPress dependency):

```php
use WP3DCONF\Formula\Expression;

$parsed = Expression::parse( '{width} * {height} / 1000000 * 45' );
// array( 'ok' => true, 'errors' => array(), 'names' => array( 'width', 'height' ), 'node' => … )

if ( ! $parsed['ok'] ) {
	$error = $parsed['errors'][0]; // array( 'code' => 'syntax', 'message' => 'A bracket is never closed.', 'pos' => 12 )
}

Expression::evaluate( '{width} * {height} / 1000000 * 45', array( 'width' => 1200, 'height' => 1500 ) ); // 81.0
```

`names` lists the variables the expression uses, so you can check each has a row in `formula.variables`.
Error codes are `empty`, `too_long`, `syntax`, `unknown_function`, `bad_arity` and `too_deep`; messages are
English and not translated.

**Saving.** Don't run an expression through `sanitize_text_field()`: it strips `<…>` as a tag and `%`
followed by two hex digits, so `{a}<5 and {b}>2` would save as `{a}2`. Writing the settings with
`update_post_meta()` is safe while the module is enabled — `sanitize_post_meta__wp3dconf_layer_settings`
sanitises every expression. To sanitise yourself, use `Expression::sanitize( $expression )`, which keeps
the language's characters and turns anything else into `?`, so a bad character fails to parse rather than
pricing something else.

---

## 6. Customising without editing the module

### 6.1 Overriding templates from a theme

The template loads through `wp3dconf_layer_template()` and the plugin's template loader, so it can be
overridden under the active theme:

```
{theme}/wp3dconf/wp-3d-configurator/layer/layer-formula.php
```

It receives `$args['formula']` with `uid`, `label` and `price_mode`. Default output in price mode:

```html
<div class="wp3dconf-formula">
  <span class="wp3dconf-formula-label">Glass</span>
  <span class="wp3dconf-price" x-text="$store.wp3dconf_123.getFormulaPrice( &quot;Fg7a2c&quot; )"></span>
</div>
```

In value mode the second span is `<span class="wp3dconf-formula-result" x-text="….getFormulaResult( … )">`.
The label span is left out when the label is empty.

Rules an override must respect:

- Bind with `x-text`, never `x-html`. The output passes through `wp_kses()`, which drops `x-html`
  silently; that's why `getFormulaPrice()` returns plain text.
- Escape every echoed value at the point of output.

### 6.2 Styling

`assets/frontend/formula.css`. Override from your theme with more specific selectors:

| Selector | Element |
|---|---|
| `.wp3dconf-control-item.wp3dconf-layer-type-formula` | The list item (`wp3dconf-layer-hidden` / `-disabled` added by Conditional Logic) |
| `.wp3dconf-formula` | The row: label left, figure right |
| `.wp3dconf-formula-label` | The label |
| `.wp3dconf-formula-result` | A value-mode result |
| `.wp3dconf-formula .wp3dconf-price` | A price-mode charge |

---

## 7. Debugging

| Symptom | Check |
|---|---|
| A formula reads 0 | A syntax error, an unknown or deleted variable, an unchosen layer, or a loop between formulas. Check the expression with `Expression::parse()`. |
| Live price and cart price differ | A price change made on one side only. Mirror it in `manipulate_layer_price` and `price/layer_amounts`. |
| My `layer_amounts` change has no effect | It ran before the module's own filter. Register at a priority above 10. |
| An imported expression lost characters | It went through `sanitize_text_field()`, or was saved while the module was off. Use `Expression::sanitize()`. |
| A hidden formula is still charged | By design: the server never evaluates Conditional Logic. Put the condition in the expression with `if()`. |
| A value-mode result prints in the cart though *Show the result* is off | Known limitation: only the frontend row and summary popup honour `formula_show`. |
| A formula reading a Color Picker's price is 0 in the cart | Known limitation: the server prices a chosen Color Picker at 0 while the browser counts it. |
| Rows render but prices never update | The configurator has no 3D model, so no frontend scripts load. |
| Nothing renders in a theme override | It binds with `x-html`; use `x-text`. |
