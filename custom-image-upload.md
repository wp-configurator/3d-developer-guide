# Custom Image Upload - Developer Guide

**Audience:** developers extending or integrating the Custom Image Upload layer: themes and add-on
plugins.
**Not for:** shop owners (plugin settings) or 3D artists (see
[custom-image-upload-blender-guide.md](custom-image-upload-blender-guide.md)).

Read [extend-modules.md](extend-modules.md) first for the general module contract. This guide covers
only what is specific to this module and where you can hook into it.

---

## 1. What it does

A **Custom Image Upload layer** is a layer type in the Model Structure tab. A shopper picks an image,
positions it in a popup (pan, zoom, rotate, align, optional SVG mask), and on **Apply** the result is
composited to a 1024px PNG and used as the **whole texture map** of one or more target meshes.

Nothing is projected onto the model. The target mesh's UVs decide where the image lands, which is why
the model must be prepared for it (see the Blender guide).

```
file picked --> popup (canvas 400px preview) --Apply--> bake to 1024px PNG
                                                          |
             viewer.swapTexture(meshNames, dataUrl, {flipY:false})   <- shopper sees it now
                                                          |
     background upload x2 (original, baked) --> media library attachments
                                                          |
     store.customData[uid] = { original_attachment_id, baked_attachment_id, transform }
```

Two uploads happen, both only on Apply (never while the shopper is trying photos):

| `kind` | Content | Used for |
|---|---|---|
| `original` | The untouched file the shopper picked | Re-editing (`openForEdit()`), quote/PDF "Original image" link |
| `baked` | The masked/clipped 1024px PNG | Texture on the mesh after reload, cart/summary thumbnail |

---

## 2. File map

```
modules/custom-image-upload/
├── class-custom-image-upload.php   # Module_Base subclass: layer type, settings, AJAX, summaries, validation
├── class-mask.php                  # SVG -> path data + viewBox
└── assets/frontend/
    ├── custom-image-upload.css
    └── js/                         # unbundled native ES modules (no build step)
templates/
├── layer/layer-custom-image-upload.php   # per-layer trigger (upload button, help, format notice)
└── custom-image-upload-popup.php         # the shared crop popup, printed once per configurator
```

**There is no Vite build for this module.** Unlike other modules, the JS is hand-authored and served
as-is, so edit files under `assets/frontend/js/` directly and reload. There is no editor bundle either:
the layer's settings come from PHP (`layer_settings_controls()`).

### JS modules (one concern each)

| File | Responsibility |
|---|---|
| `constants.js` | Tunables: `CANVAS_SIZE` 400, `BAKE_SIZE` 1024, `PRINT_ZONE_FILL`, trim thresholds |
| `runtime.js` | Shared mutable state `rt` and `createState()` |
| `utils.js` | `getLayerSettings()`, `normalizeMeshNames()`, `i18n()`, `formatString()`, `clamp()` |
| `mask.js` | `getMask()`, `drawImageOnto()`, `drawMaskOverlay()` |
| `content-box.js` | `contentBox()`: the artwork's opaque bounds (for auto-fit and spill warning) |
| `zones.js`, `reference-zones.js` | Print zone measured off the model's existing texture |
| `fit.js`, `align.js`, `controls.js` | Starting fit, zoom bounds, alignment buttons, slider sync |
| `render.js` | `draw()` (preview) and `bakeToDataUrl()` (final PNG) |
| `session.js` | `openFor()`, `openForEdit()`, `autofit()`, `apply()`, `removeImage()` |
| `upload.js` | `postUpload()`, `uploadOriginal()`, `rememberAttachmentUrl()` |
| `restore.js` | `restoreBakedTextures()`: reapply saved bakes on model load |
| `summary.js` | Summary popup filters |
| `trigger-ui.js`, `trigger-events.js`, `popup-ui.js`, `pointer.js`, `guide.js` | DOM wiring |
| `init.js` | Registers everything; entry via `main.js` |

### The dependency list is load-bearing

Files import each other through bare specifiers such as
`import { rt } from '@wp3dconf/custom-image-upload/runtime'`. WordPress resolves these from the page's
import map, and only adds a file to that map if it is reachable through the dependency list in
`Custom_Image_Upload::frontend_script_modules()`.

> **If you add, remove or change an `import`, update `frontend_script_modules()` to match.** A missing
> entry fails in the browser as *"failed to resolve module specifier"*, not in PHP.

Only the `main` entry is enqueued; everything else is registered and pulled in as a dependency.
File names must match `^[a-z0-9_-]+$` and the file must exist in `assets/frontend/js/`.

---

## 3. Data model

### 3.1 Layer settings (saved with the configurator)

Stored in `_wp3dconf_layer_settings` under the layer's `uid`. Frontend reads them at
`window.WP3dConf.layers[uid].settings`.

| Key | Type | Default | Notes |
|---|---|---|---|
| `custom_image_upload_target_mesh` | `string[]` (legacy: `string`) | none | Mesh names to texture. Always read through `normalizeMeshNames()` / accept both shapes |
| `custom_image_upload_mask_svg` | attachment ID | none | Admin-chosen SVG |
| `custom_image_upload_mask_path` | string | derived | SVG path `d`; written by `store_mask_path()` on save. **Do not set by hand** |
| `custom_image_upload_mask_view_box` | string | derived | SVG viewBox; same rule |
| `custom_image_upload_guidelines` | string | `''` | Shown in the help panel |
| `custom_image_upload_allowed_formats` | `string[]` (legacy: comma string) | `jpg,png,webp` | Options: `jpg`, `png`, `webp`, `gif`. Use `Custom_Image_Upload::parse_allowed_formats()` |
| `custom_image_upload_max_size` | number (MB) | `5` | |
| `custom_image_upload_required` | bool | `false` | Enforced server-side only |

Older configurators saved a single mesh string and a comma-separated format string. Both shapes must
keep loading, so always go through the two normalisers above rather than reading the raw value.

### 3.2 Custom data (per shopper selection)

`store.customData[uid]` on the frontend, `custom_data` in cart/quote payloads:

```json
{
  "original_attachment_id": 812,
  "baked_attachment_id": 813,
  "transform": { "x": 12, "y": -5, "scale": 1.5, "rotation": 15 }
}
```

- `baked_attachment_id` is the internal name for what shoppers and summaries call the **edited** image.
- Both IDs start `null` at Apply and are filled in as each upload resolves (`updateCustomData()` merges).
- A layer counts as **fulfilled** only once `baked_attachment_id` is set. An original with no bake does
  not satisfy `custom_image_upload_required`.
- `transform` is in **preview-canvas pixels** (400px space). `bakeToDataUrl()` scales x, y and scale by
  `BAKE_SIZE / CANVAS_SIZE` when compositing.

### 3.3 Frontend globals

| Global | Content |
|---|---|
| `window.WP3dConf.customImageUpload.i18n` | Translated strings (from `frontend_payload()`) |
| `window.WP3dConf.customImageUpload.printZones` | Diagnostic function; logs measured zones |
| `window.WP3dConf.attachmentUrls` | `{ [attachmentId]: url }`. Extended at runtime by `rememberAttachmentUrl()` |
| `window.WP3dConf.viewer` | The three.js viewer (`swapTexture`, `resetTexture`) |
| `window.WP3dConf.layers`, `.nonce`, `.ajaxUrl`, `.postId` | Core payload |

---

## 4. Extension points

### 4.1 Adding a similar layer type

Copy the pattern from `Custom_Image_Upload`, all in `Module_Base` overrides:

| Method | What this module returns |
|---|---|
| `get_slug()` | `'custom-image-upload'` |
| `layer_type()` | `array( 'slug', 'name', 'icon' )` for the Model Structure list |
| `layer_settings_controls( $settings )` | Adds a settings section, gated by `array( 'terms' => array( array( 'layer_type', '<slug>', '===' ) ) )` |
| `frontend_payload( $post_id )` | Becomes `window.WP3dConf.<camelSlug>` |
| `init()` | Extra hooks |

Do not rename or change the return contract of `Module_Base` methods; add-on modules in other
repositories depend on them.

### 4.2 PHP hooks this module uses (and you can also use)

| Hook | Type | Args | Purpose here |
|---|---|---|---|
| `wp3dconf/frontend/controls/layer_custom-image-upload_html` | action | `$layer`, `$skin` | Renders the trigger via `wp3dconf_layer_template()` |
| `wp3dconf/frontend/after_skin_display` | action | `$skin` | Prints the popup once |
| `wp3dconf/editor/save_configurator` | action | `$post_id` | Priority 20: extracts mask path after core saved settings |
| `wp3dconf/frontend/data` | filter | `$data`, `$post_id` | Adds attachment URLs for saved images |
| `wp3dconf/data/custom_data_string` | filter | `$string`, `$uid`, `$custom_data`, `$data` | Plain-text summary (cart, email, PDF) |
| `wp3dconf/data/custom_data_html` | filter | `$html`, `$uid`, `$item` | Original/Edited image links |
| `wp3dconf/data/custom_data_layer_price` | filter | `$price`, `$layer`, `$custom_data`, `$data` | Returns the layer's configured price |
| `wp3dconf/quote/validation_errors` | filter | `$errors` (`WP_Error`), `$args` | Required-layer check for quotes |
| `woocommerce_add_to_cart_validation` | filter | `$passed`, `$product_id` | Same check for WooCommerce |
| `wp3dconf/utils/allowed_html_tags` | filter | `$tags` | Re-adds `button`, `p`, `label`, `br` to the control-item kses list |
| `wp3dconf/frontend/script_modules` | filter | `$modules`, `$slug` | Change the JS file/dependency map |

**Adjusting the quote/email output** from your own plugin: filter `wp3dconf/data/custom_data_html`
at a priority above 10 and check `$item['type'] === 'custom-image-upload'`. Escape everything you
add: this HTML reaches email bodies and the PDF. Use `<a href>` with `esc_url()`; do not emit `<img>`
or `style` (the WooCommerce Cart block re-sanitises client-side and strips both).

```php
add_filter( 'wp3dconf/data/custom_data_html', function ( $html, $uid, $item ) {
	if ( empty( $item['type'] ) || 'custom-image-upload' !== $item['type'] ) {
		return $html;
	}

	$id  = (int) ( $item['custom_data']['baked_attachment_id'] ?? 0 );
	$url = $id ? wp_get_attachment_url( $id ) : '';

	return $url
		? $html . ' <a href="' . esc_url( $url ) . '" download>' . esc_html__( 'Download print file', 'my-addon' ) . '</a>'
		: $html;
}, 20, 3 );
```

**Changing the layer price rule:** filter `wp3dconf/data/custom_data_layer_price` (return
`array( 'price' => float, 'sale_price' => float )`). Core zeroes a layer's price the moment it has
custom data, so removing this module's filter would make the layer free once an image is uploaded.

**Extra validation** (e.g. minimum resolution on the server): hook
`wp3dconf/quote/validation_errors` and `woocommerce_add_to_cart_validation`. Decode submitted values
with `Configurator_Utils::decode_custom_data( $custom_data )`, which sanitises per value. Both
entry points must be covered; the quote hook alone does not guard add-to-cart.

### 4.3 JS hooks this module uses (and you can also use)

Register inside the `wp3dconf/frontend:ready` event (see [hooks-frontend.md](hooks-frontend.md)).

| Hook | Type | Payload | Purpose here |
|---|---|---|---|
| `wp3dconf/frontend/afterModelLoaded` | action | `{ store }` | Measure reference zones, then restore saved bakes |
| `wp3dconf/frontend/summary/layerCustomData` | filter | `data`, `{ uid, store }` | Replaces `[object Object]` for `transform` with readable text |
| `wp3dconf/frontend/summary/layerThumbnail` | filter | `thumbnail`, `{ uid, store }` | Baked image as CSS `background-image` |

Both summary filters must return their first argument untouched for layers that are not
`custom-image-upload` (check `store.layers[uid].type`).

**Ordering constraint:** in `afterModelLoaded`, `captureReferenceZones()` must run **before**
`restoreBakedTextures()`. `swapTexture()` disposes the texture it replaces, and the reference-zone
measurement reads that original texture. If your add-on also calls `swapTexture()` on the same mesh in
this action, run at a later priority than 10.

### 4.4 Driving the layer from your own code

The shopper's selection lives in the frontend store passed to
`afterModelLoaded` (`payload.store`):

| Call | Effect |
|---|---|
| `store.activateLayer(uid, customData)` | Marks the layer active and stores `customData[uid]` (replaces) |
| `store.updateCustomData(uid, partial)` | Merges into `customData[uid]` |
| `store.deactivateLayer(uid)` | Deactivates and deletes `customData[uid]` |

And the viewer (`window.WP3dConf.viewer`):

| Call | Effect |
|---|---|
| `swapTexture(meshNames[], urlOrDataUrl, { flipY: false })` | Replaces the mesh material's `map`, disposing the old one |
| `resetTexture(meshNames[])` | Restores the map/colour captured on first swap |

Use **`{ flipY: false }`** for anything shaped like the baked PNG (top-down, matches glTF UVs).
Omitting it renders the image upside down while its thumbnail looks right.

Example: apply an image programmatically, exactly as `apply()` does after a bake:

```js
window.addEventListener('wp3dconf/frontend:ready', () => {
  window.WP3dConf.addAction('wp3dconf/frontend/afterModelLoaded', ({ store }) => {
    const uid = 'YOUR_LAYER_UID';
    const meshes = window.WP3dConf.layers[uid].settings.custom_image_upload_target_mesh;

    window.WP3dConf.viewer.swapTexture([].concat(meshes), 'https://example.com/preset.png', { flipY: false });
    store.activateLayer(uid, { original_attachment_id: null, baked_attachment_id: null, transform: { x: 0, y: 0, scale: 1, rotation: 0 } });
  }, 20);
});
```

> An image applied this way is visual only. Unless it also becomes an attachment referenced by
> `baked_attachment_id`, it will not survive reload, appear in the quote, or satisfy
> `custom_image_upload_required`. To persist, POST to the AJAX endpoint (section 5).

---

## 5. The upload endpoint

`wp_ajax_wp3dconf_upload_custom_image` and `wp_ajax_nopriv_wp3dconf_upload_custom_image`
(`Custom_Image_Upload::upload_image()`).

**Who can call it:** anyone with a valid `wp3dconf_nonce`, including logged-out visitors. There is no
capability check, by design: shoppers are anonymous. The nonce proves the request came from a
configurator page, not that the visitor is trusted. There is currently **no rate limit**.

### Request (`POST`, form-encoded)

| Field | Required | Description |
|---|---|---|
| `action` | yes | `wp3dconf_upload_custom_image` |
| `nonce` | yes | `WP3dConf.nonce` (`wp3dconf_nonce`) |
| `config_id` | yes | Configurator post ID (`WP3dConf.postId`) |
| `uid` | yes | Layer uid; settings are looked up from it |
| `image_data` | yes | Data URI: `data:image/<type>;base64,<payload>` |
| `kind` | no | `baked` or `original` (default). Label only; both take the same path |
| `transform` | no | JSON `{x,y,scale,rotation}`, echoed back sanitised |

### Response

```json
{ "success": true, "data": { "kind": "baked", "attachment_id": 813, "url": "https://.../wp3dconf-custom-image-....png", "transform": { "x": 0, "y": 0, "scale": 1, "rotation": 0 } } }
```

On failure: `{ "success": false, "data": { "message": "..." } }` with a translated message.

### Server-side validation (in order)

1. Nonce, then required fields (`config_id`, `uid`, `image_data`).
2. Data URI must match `data:image/...;base64,`. The declared mime is **not trusted**.
3. Decoded size against the layer's `max_size` (MB), measured **after** decoding.
4. `getimagesizefromstring()` on the bytes; unreadable = rejected.
5. Width and height each at most `Custom_Image_Upload::MAX_DIMENSION` (6000px).
6. Extension derived from the **sniffed** mime (`jpg`, `png`, `webp`, `gif` only) must be in the layer's
   allowed formats.

Saved via `wp_upload_bits()` + `wp_insert_attachment()` as
`wp3dconf-custom-image-<configId>-<uid>-<kind>-<random>.<ext>`, with thumbnail generation
suppressed, and marked `_wp3dconf_temp_image` (a timestamp).

---

## 6. Customising without editing the module

### 6.1 Overriding templates from a theme

Both templates go through `Frontend_Template`, so they can be overridden under the active theme:

```
{theme}/wp3dconf/wp-3d-configurator/layer/layer-custom-image-upload.php
{theme}/wp3dconf/wp-3d-configurator/custom-image-upload-popup.php
```

Rules an override must respect:

- **Keep the class names and `data-*` attributes.** The JS binds to them:
  `.wp3dconf-custom-image-upload[data-uid]`, `-trigger`, `-input`, `-replace`, `-remove`, and in the
  popup `-canvas`, `-zoom`, `-rotate`, `-apply`, `-cancel`, `-error`, `-warning`,
  `.wp3dconf-custom-image-upload-align-btn[data-axis][data-position]`.
- `init.js` requires the popup root, canvas, zoom and rotate inputs, Apply and Cancel. **Auto Fit, the
  zoom/rotate number fields and the backdrop are optional**; an override without them still works.
- The layer template's output passes through `wp_kses()`. `button`, `p`, `label`, `br` survive only
  because of `Custom_Image_Upload::allowed_html_tags()`. **Anything else you add is stripped silently,
  keeping only its text.** Widen with `wp3dconf/utils/allowed_html_tags` rather than editing the
  call site, and never allow event-handler attributes or Alpine directives beyond what the core list
  already permits.
- Escape every echoed value at the point of output.

### 6.2 Styling

`assets/frontend/custom-image-upload.css`. Override from your theme with more specific selectors;
avoid editing the module file.

### 6.3 Changing strings

Frontend strings come from `frontend_payload()['i18n']` and are read in JS through `i18n()`. Translate
them with a translation plugin or `.po` files for the plugin's text domain. The module has no filter
for individual strings, so add one to `frontend_payload()` if you need it. Do not hardcode text in the JS.

## 7. Debugging

| Symptom | Check |
|---|---|
| Blank page / *failed to resolve module specifier* | `import` added without updating `frontend_script_modules()` |
| Trigger shows label text but no button or paragraph | kses stripped an element; check `allowed_html_tags()` |
| Image lands upside down on the model | `swapTexture()` called without `{ flipY: false }` |
| Image fine in popup, wrong place on model | Mesh UVs (Blender guide), not the plugin |
| "Edit Image" click does nothing | `attachmentUrls` has no entry for `original_attachment_id` (`rememberAttachmentUrl()` not called for a new upload) |
| Texture gone after reload, cart still shows image | `baked_attachment_id` was `null` (tab closed before upload resolved) |
| `toDataURL` / tainted canvas error on production | `crossOrigin` missing on a CDN-served image |
| Apply shows "not set up correctly" | Layer has no Target Mesh configured |
| Nothing saved to cart though texture shows | `apply()` ran before `afterModelLoaded` populated `rt.storeRef` |
| Existing configurator lost its mesh/format after update | Legacy string shape read without the normalisers |

In the console, `WP3dConf.customImageUpload.printZones()` reports the print zone measured for each
layer, useful when auto-fit or the guide preview looks wrong.
