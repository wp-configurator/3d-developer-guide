# Custom Text - Developer Guide

**Audience:** developers extending or integrating the Custom Text layer: themes and add-on plugins.
**Not for:** shop owners (see [custom-text.md](custom-text.md)) or 3D artists (see
[custom-text-blender-guide.md](custom-text-blender-guide.md)).
**Applies to:** plugin v1.3.6
**Last updated:** 2026-10-30

Read [extend-modules.md](extend-modules.md) first for the general module contract. This guide covers
only what is specific to this module and where you can hook into it. For a comparable module, see
[custom-image-upload.md](custom-image-upload.md).

> **Pro only.** `custom-text` is in `build-scripts/build-lite.php`'s strip list and required from
> `pro/class-wp3dconf.php`.

---

## 1. What it does

A **Custom Text layer** is a layer type in the Model Structure tab. A shopper types text, optionally
picks a font and style in a popup, and the text is drawn onto a 1024px transparent canvas that
replaces the **whole texture map** of one or more target meshes.

Nothing is uploaded. The text and its style are saved as plain values in the layer's custom data,
and the texture is drawn again from them whenever the model loads. There is no AJAX endpoint.

```
keystroke --> normalizeText() --150ms debounce--> render 1024px canvas
                                                      |
        viewer.swapTexture(meshNames, dataUrl, {flipY:false})   <- shopper sees it now
                                                      |
   store.activateLayer / updateCustomData(uid, { text, font, ...style })
                                                      |
   cart / quote post custom data --> server cleans it again --> price, summary, validation
```

**The server never trusts the browser.** Every value is cleaned again on the server by the same rules
(`clean_entry()`) before it is priced, summarised or checked.

---

## 2. File map

```
modules/custom-text/
├── class-custom-text.php        # Module_Base subclass: layer type, settings, fonts, cleaning, price, validation
├── google-fonts.php             # Family names offered in the Fonts setting
└── assets/frontend/
    ├── custom-text.css
    └── js/                      # unbundled native ES modules (no build step)
templates/
├── layer/layer-custom-text.php  # per-layer text field and Customize button
└── custom-text-popup.php        # the shared Customize popup, printed once per configurator
```

**There is no Vite build for this module.** The JS is hand-authored and served as-is, so edit files
under `assets/frontend/js/` and reload. There is no editor bundle either: the layer's settings come
from PHP (`layer_settings_controls()`).

### JS modules (one concern each)

| File | Responsibility |
|---|---|
| `constants.js` | Tunables: `TEXTURE_SIZE` 1024, `UPDATE_DELAY` 150, ranges, edge margin |
| `runtime.js` | Shared mutable state `rt` |
| `utils.js` | `getLayerSettings()`, `normalizeMeshNames()`, `i18n()`, `clamp()` |
| `rules.js` | `getRules()`, `normalizeText()`, `normalizeStyle()`, `normalizeFont()` |
| `fonts.js` | `loadFont()`: loads a downloaded font file as a `FontFace` |
| `content-box.js`, `zone.js`, `reference-zones.js` | Measure the print zone off the mesh's existing texture |
| `layout.js`, `render.js` | Fit the text into the zone; `drawText()`, `renderTextToDataUrl()` |
| `apply.js` | `applyText()`, `saveText()`, `scheduleApply()`, `clearLayer()` |
| `choices.js` | The shopper's remembered font and style per layer |
| `preview.js`, `popup.js`, `popup-events.js` | The Customize popup |
| `restore.js` | `restoreTexts()`: redraw saved text on model load |
| `events.js` | Typing, Customize click, layer deactivation |
| `price.js` | Live price per character |
| `summary.js` | Summary popup wording |
| `init.js` | Registers everything; entry via `main.js` |

### The dependency list is load-bearing

Files import each other through bare specifiers such as
`import { rt } from '@wp3dconf/custom-text/runtime'`. WordPress resolves these from the page's
import map, and only adds a file to that map if it is reachable through the dependency list in
`Custom_Text::frontend_script_modules()`.

> **If you add, remove or change an `import`, update `frontend_script_modules()` to match.** A missing
> entry fails in the browser as *"failed to resolve module specifier"*, not in PHP.

You can also change the map from outside with the `wp3dconf/frontend/script_modules` filter
(`$modules`, `$slug`).

---

## 3. Data model

### 3.1 Layer settings (saved with the configurator)

Stored in `_wp3dconf_layer_settings` under the layer's `uid`. The frontend reads them at
`window.WP3dConf.layers[uid].settings`.

| Key | Type | Default | Notes |
|---|---|---|---|
| `custom_text_target_mesh` | `string[]` (legacy: `string`) | none | Read through `Custom_Text::parse_target_meshes()` / `normalizeMeshNames()` |
| `custom_text_input_mode` | `text` \| `name` \| `number` \| `alphanumeric` | `text` | |
| `custom_text_default_text` | string | `''` | |
| `custom_text_placeholder` | string | `''` | |
| `custom_text_max_length` | int | `30` | 1-100; `0`/empty means 30 |
| `custom_text_letter_case` | `none` \| `uppercase` \| `lowercase` | `none` | |
| `custom_text_color` | hex | `#000000` | Starting text colour |
| `custom_text_fonts` | `string[]` | `[]` | Families from `font_list()` |
| `custom_text_default_font` | string | `''` | Must also be in `custom_text_fonts` |
| `custom_text_curve` | number | `0` | -100 to 100 |
| `custom_text_letter_spacing` | number | `0` | -10 to 50 |
| `custom_text_outline_width` | number | `0` | 0 to 20 |
| `custom_text_outline_color` | hex | `#ffffff` | |
| `custom_text_controls` | `string[]` | `[]` | Subset of `Custom_Text::CONTROLS`: `color`, `curve`, `spacing`, `rotation`, `outline`, `size`, `position` |
| `custom_text_price_per_character` | number | `0` | Spaces are not charged |
| `custom_text_required` | bool | `false` | Enforced server-side only |

Layers saved before a setting existed simply lack the key, so every reader falls back to the default.
A layer saved before the `color` control existed does not list it in `custom_text_controls`, so its
shoppers cannot change the colour until an admin enables it.

### 3.2 Custom data (per shopper selection)

`store.customData[uid]` on the frontend, `custom_data` in cart/quote payloads. It is flat:

```json
{
  "text": "ALEX",
  "font": "Pacifico",
  "color": "#ff0000",
  "curve": 20,
  "spacing": 0,
  "rotation": 0,
  "outline": 2,
  "outlineColor": "#ffffff",
  "size": 100,
  "positionX": 0,
  "positionY": 0
}
```

- A style value the shopper may not change (not in `custom_text_controls`) is always the layer's own
  default, whatever was submitted.
- `font` is omitted when the layer offers no fonts.
- An empty text deactivates the layer.
- A layer counts as filled in only if its text is non-empty **after** the layer's rules are applied.
  A name layer holding only digits is empty.

### 3.3 Style ranges

| Value | Range | Needs control |
|---|---|---|
| `curve` | -100 to 100 | `curve` |
| `spacing` | -10 to 50 | `spacing` |
| `rotation` | -180 to 180 | `rotation` |
| `outline` | 0 to 20 | `outline` |
| `outlineColor` | hex | `outline` |
| `size` | 25 to 100 | `size` |
| `positionX`, `positionY` | -100 to 100 | `position` |
| `color` | hex | `color` |

`Custom_Text::STYLE_RANGES` and `assets/frontend/js/constants.js` hold the same numbers. **Change both.**

### 3.4 Frontend globals

| Global | Content |
|---|---|
| `window.WP3dConf.customText.i18n` | Translated strings (from `frontend_payload()`) |
| `window.WP3dConf.customText.fonts` | `{ [family]: url }` for every downloaded font a layer offers |
| `window.WP3dConf.viewer` | The three.js viewer (`swapTexture`, `resetTexture`, `findMeshes`) |
| `window.WP3dConf.layers` | Layer settings, `.nonce`, `.ajaxUrl`, `.postId` |

---

## 4. Server-side pipeline

The same helpers run for the cart, quotes, emails, PDFs and validation, so the figure a shopper sees
is the figure they pay.

| Method | Purpose |
|---|---|
| `clean_text( $value, $settings )` | Applies input type, letter case and max length. Non-UTF-8 gives `''` |
| `clean_style( $entry, $settings )` | Keeps each allowed value inside its range; everything else is the layer default |
| `clean_entry( $entry, $settings )` | `{ text, font, style }`. The font must be one the layer selected |
| `style_defaults( $settings )` | What the layer looks like before the shopper changes anything |
| `allowed_controls( $settings )` | The controls the admin gave the shopper |
| `character_count( $text )` | Characters charged for (no whitespace) |
| `style_summary( $style, $neutral_color )` | Labelled values that differ from neutral |
| `missing_from_layers( $layers, $custom_data, $unavailable )` | Required layers left empty |

These are `public static`, so you can call them from your own code. They are not part of the
`Module_Base` contract, but treat their signatures as stable.

### Prices

`wp3dconf/data/custom_data_layer_price` is how a layer is priced. Core zeroes a layer's price the
moment it has custom data, so this module's filter returns:

```
price      = layer price + price per character x characters
sale_price = sale price + the same per-character charge (0 when there is no sale price)
```

A layer with no usable text is not charged. The frontend (`price.js`) computes the same sum for the
live total through `wp3dconf/frontend/price/layer_amounts`.

---

## 5. Extension points

### 5.1 PHP filters added by this module

All names start with `wp3dconf/custom_text/`.

| Filter | Args | Use it to |
|---|---|---|
| `frontend_payload` | `$payload`, `$post_id` | Change the data printed as `window.WP3dConf.customText` |
| `layer_settings` | `$section` | Add, remove or change a setting in the "Custom Text" panel section |
| `font_list` | `$families` | Add or remove selectable Google Fonts. Names that are not plain family names are dropped afterwards |
| `offered_fonts` | `$offered`, `$settings` | Change the fonts a layer offers a shopper |
| `default_font` | `$font`, `$settings`, `$offered` | Change the font used until the shopper picks one |
| `layer_font_urls` | `$urls`, `$post_id` | Change the font file URLs sent to the frontend (e.g. serve them from a CDN) |
| `max_font_downloads` | `$limit`, `$post_id` | Change the fonts downloaded per save (default 8) |
| `download_font` | `$allow`, `$family`, `$path` | Return `false` to stop the Google request |
| `clean_text` | `$text`, `$value`, `$settings` | Change the cleaned text |
| `clean_style` | `$style`, `$entry`, `$settings` | Change a layer's cleaned style |
| `clean_entry` | `$clean`, `$entry`, `$settings` | Change the full cleaned `{ text, font, style }` |
| `style_defaults` | `$defaults`, `$settings` | Change the starting style |
| `allowed_controls` | `$allowed`, `$settings` | Change which controls a shopper gets. Only values of `CONTROLS` are kept |
| `character_count` | `$count`, `$text` | Change how many characters are charged for |
| `layer_price` | `$price`, `$layer`, `$text`, `$settings` | Change the price of a layer that has text |
| `summary_labels` | `$labels` | Rename the labels in the cart, quote, email and PDF |
| `style_summary` | `$rows`, `$style` | Change the style rows listed in a summary |
| `summary_parts` | `$parts`, `$entry`, `$uid` | Change the parts of the one-line summary |
| `missing_layers` | `$missing`, `$layers`, `$values`, `$unavailable` | Change the required layers reported as empty |

### 5.2 PHP actions added by this module

| Action | Args | Fires |
|---|---|---|
| `wp3dconf/custom_text/before_popup` | `$skin` | Before the Customize popup is printed |
| `wp3dconf/custom_text/after_popup` | `$skin` | After the Customize popup is printed |
| `wp3dconf/custom_text/font_downloaded` | `$family`, `$path` | After a font file is saved to the site |

### 5.3 Core hooks this module uses (and you can also use)

| Hook | Type | Args | Purpose here |
|---|---|---|---|
| `wp3dconf/frontend/controls/layer_custom-text_html` | action | `$layer`, `$skin` | Renders the field via `wp3dconf_layer_template()` |
| `wp3dconf/frontend/after_skin_display` | action | `$skin` | Prints the popup once |
| `wp3dconf/editor/save_configurator` | action | `$post_id` | Priority 20: downloads selected fonts after core saved the settings |
| `wp3dconf/data/custom_data_string` | filter | `$string`, `$uid`, `$custom_data`, `$data` | One-line summary |
| `wp3dconf/data/custom_data_layer_price` | filter | `$price`, `$layer`, `$entry`, `$data` | Layer price |
| `wp3dconf/quote/validation_errors` | filter | `$errors`, `$args` | Required-layer check for quotes |
| `woocommerce_add_to_cart_validation` | filter | `$passed`, `$product_id` | Same check for WooCommerce |
| `wp3dconf/utils/allowed_html_tags` | filter | `$tags` | Adds `button` to the control-item kses list |
| `wp3dconf/frontend/script_modules` | filter | `$modules`, `$slug` | Change the JS file and dependency map |

### 5.4 JS hooks this module uses

Register inside the `wp3dconf/frontend:ready` event (see [hooks-frontend.md](hooks-frontend.md)).

| Hook | Type | Payload | Purpose here |
|---|---|---|---|
| `wp3dconf/frontend/afterModelLoaded` | action | `{ store }` | Measure print zones, then restore saved text |
| `wp3dconf/frontend/layer/deactivated` | action | `{ uid, layer }` | Reset a text layer that something else (e.g. Conditional Logic) deselected |
| `wp3dconf/frontend/price/layer_amounts` | filter | `amounts`, `{ uid, layer, store }` | Add the per-character price to the live total |
| `wp3dconf/frontend/summary/layerCustomData` | filter | `data`, `{ uid, store }` | Readable wording in the summary popup |

Filters must return their first argument untouched for layers that are not `custom-text`.

**Ordering constraint.** In `afterModelLoaded`, `captureReferenceZones()` must run **before**
`restoreTexts()`. `swapTexture()` disposes the texture it replaces, and the print-zone measurement
reads that original texture. If your add-on also calls `swapTexture()` on the same mesh in this
action, register it at a later priority.

---

## 6. Recipes

### 6.1 Limit the choice of fonts to a short list

```php
add_filter( 'wp3dconf/custom_text/font_list', function ( $families ) {
	return array( 'Roboto', 'Pacifico', 'Bebas Neue' );
} );
```

### 6.2 Stop the plugin contacting Google

Serve fonts yourself, or turn the download off. Fonts already on the site keep working.

```php
add_filter( 'wp3dconf/custom_text/download_font', '__return_false' );
```

To supply your own files, copy a `.woff2` to the path `Custom_Text::font_path( $family )` returns
(`uploads/wp3dconf-fonts/<family-slug>.woff2`), or react to `font_downloaded` to sync new files to
your CDN.

### 6.3 Block words

Do it in `clean_text` so the browser-posted value, the price and the summary all agree.

```php
add_filter( 'wp3dconf/custom_text/clean_text', function ( $text ) {
	$blocked = array( 'badword' );

	return in_array( strtolower( $text ), $blocked, true ) ? '' : $text;
} );
```

An empty result makes a required layer fail validation and a normal layer free, which is the
correct behaviour for text you refuse to print. Block the same words in the browser if you want
to tell the shopper why (see 6.8), since this only affects the server's copy.

### 6.4 Charge a flat fee beyond ten characters

```php
add_filter( 'wp3dconf/custom_text/layer_price', function ( $price, $layer, $text ) {
	if ( mb_strlen( $text ) > 10 ) {
		$price['price'] += 2.50;
	}

	return $price;
}, 10, 3 );
```

The frontend total will not include this unless you add a matching callback to
`wp3dconf/frontend/price/layer_amounts`. A mismatch shows the shopper a different figure from the
cart. Keep them in step.

### 6.5 Take a control away from shoppers

```php
add_filter( 'wp3dconf/custom_text/allowed_controls', function ( $allowed ) {
	return array_values( array_diff( $allowed, array( 'rotation' ) ) );
} );
```

This filter only runs on the server. It decides what the server accepts, and it controls whether
the Customize button is printed. The popup in the browser reads `custom_text_controls` from the
layer's saved settings, so it can still show a control you removed here; the server then ignores
what the shopper does with it. To remove a control from the popup as well, hide its
`[data-control]` row in a template override (section 7.1). Adding a control the admin did not
tick has the opposite problem: the server accepts it but the popup does not show it.

### 6.6 Rename summary labels

```php
add_filter( 'wp3dconf/custom_text/summary_labels', function ( $labels ) {
	$labels['text'] = __( 'Engraving', 'my-addon' );

	return $labels;
} );
```

Labels are plain text and are escaped on output. Keep every key: missing keys are filled back in
from the defaults. The same list goes to `window.WP3dConf.customText.i18n.summaryLabels`, so the
summary popup matches.

### 6.7 Add a setting to the layer panel

```php
add_filter( 'wp3dconf/custom_text/layer_settings', function ( $section ) {
	$section['settings']['my_text_note'] = array(
		'name'       => esc_html__( 'Engraving note', 'my-addon' ),
		'type'       => 'text',
		'conditions' => array( 'terms' => array( array( 'layer_type', 'custom-text', '===' ) ) ),
	);

	return $section;
} );
```

Settings are saved with the layer automatically. If the setting is request-driven or lets a
shopper write something, sanitise it with the `wp3dconf/data/sanitize_layer_setting` filter.

### 6.8 Warn the shopper in the browser

```js
window.addEventListener('wp3dconf/frontend:ready', () => {
  document.addEventListener('input', (event) => {
    if (!event.target.classList.contains('wp3dconf-custom-text-input')) return;
    // validate event.target.value and show your own message
  });
});
```

The module does not offer a JS filter for the text itself. Anything that changes what is saved must
also run on the server (`clean_text`).

---

## 7. Customising without editing the module

### 7.1 Overriding templates from a theme

Both templates go through `Frontend_Template`, so they can be overridden under the active theme:

```
{theme}/wp3dconf/wp-3d-configurator/layer/layer-custom-text.php
{theme}/wp3dconf/wp-3d-configurator/custom-text-popup.php
```

Rules an override must respect:

- **Keep the class names and `data-*` attributes.** The JS binds to them:
  `.wp3dconf-custom-text[data-uid]`, `.wp3dconf-custom-text-input[data-uid]`,
  `.wp3dconf-custom-text-customize[data-uid]`, and in the popup `.wp3dconf-custom-text-popup`,
  `-popup-canvas`, `-popup-apply`, `-popup-reset`, `-popup-cancel`, `-popup-backdrop`,
  `[data-control]` rows, `[data-field]` inputs, and
  `.wp3dconf-custom-text-align-btn[data-axis][data-position]`.
- Each `[data-control]` row (`font`, `color`, `curve`, `spacing`, `rotation`, `outline`, `size`,
  `position`) is hidden or shown from the layer's rules. Leave out the rows you do not want.
- The layer template's output passes through `wp_kses()`. `button` survives only because of
  `Custom_Text::allowed_html_tags()`, and an input keeps only `type`, `class` and `data-*`.
  **Anything else is stripped silently.** Widen with `wp3dconf/utils/allowed_html_tags`, and never
  allow event-handler attributes or Alpine directives.
- Escape every echoed value at the point of output.

### 7.2 Styling

`assets/frontend/custom-text.css`. Override from your theme with more specific selectors.

### 7.3 Changing strings

Frontend strings come from `frontend_payload()['i18n']`. Use the `frontend_payload` filter to change
them, or a translation plugin / `.po` files for the plugin's text domain. Do not hardcode text in
the JS.

---

## 8. Security notes

- **No endpoint, no upload.** Custom Text has no `wp_ajax_*` handler. Its input arrives as part of
  the normal add-to-cart or quote submission.
- **Cleaned on the server.** `clean_entry()` runs for every price, summary and check. The posted
  value is a request, not a fact.
- **Escape summaries on output.** Text from `custom_data_string` and `style_summary` is plain text.
  If your add-on prints it into HTML, an email body or a PDF, escape it (`esc_html()`). A filter that
  adds markup to a summary row is your responsibility to escape.
- **Fonts.** Names come only from `font_list()` and are validated (`^[A-Za-z0-9][A-Za-z0-9 ]{0,59}$`)
  before they reach a URL or file name. Downloads go to fixed Google hosts, must be woff2 and are
  limited to 512 KB. Do not loosen these in a filter. `download_font` can only stop a download.
- **Cross-origin canvas.** The text is drawn from scratch, so the canvas is never tainted. Fonts are
  loaded from your own uploads folder; if you move them to a CDN with `layer_font_urls`, send CORS
  headers, or the `FontFace` load fails and the standard font is used.

---

## 9. Debugging

| Symptom | Check |
|---|---|
| Blank page / *failed to resolve module specifier* | An `import` was added without updating `frontend_script_modules()` |
| Field shows no Customize button | No control ticked and fewer than two fonts offered |
| Customize button label appears but nothing to click | kses stripped the button; check `allowed_html_tags()` |
| Text on the model is upside down or mirrored | The mesh's UVs (see the Blender guide), or `swapTexture()` called without `{ flipY: false }` |
| Text tiny or off-centre | Print zone could not be measured, so the 85% fallback is used; check the mesh has a transparent placeholder texture |
| Font missing for shoppers | The file is not in `uploads/wp3dconf-fonts/`; check the layer was saved and `download_font` is not blocking |
| Shopper's colour ignored on the server | `color` is not in `custom_text_controls` for the layer |
| Price differs between page and cart | A `layer_price` or `character_count` filter has no matching JS callback |
| Required layer passes with only spaces | It should not: `clean_text()` trims. Check a `clean_text` filter is not adding characters |
| Saved text gone after Conditional Logic hides the layer | By design: `layer/deactivated` resets the layer |
| Existing configurator lost its mesh after update | Legacy string shape read without `parse_target_meshes()` |
