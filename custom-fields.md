# Custom Fields

**Audience:** developers extending or integrating the Custom Fields module.
**Not for:** shop owners learning the field settings in the editor.
**Applies to:** plugin v1.4.0
**Last updated:** 2026-11-12

The Custom Fields module adds layer types that ask the customer for a value — engraving text, a note, a name, a choice from a list — rather than just letting them pick between pre-built visual options. A field's value rides through the same `customData[ uid ]` pipeline every other layer already uses, so it shows up in the summary, the quote/cart submission, the PDF invoice and the shareable link for free. This module adds **no storage of its own** and nothing to import/export: a field is a layer, and its configuration is layer settings.

---

## Overview

| Piece | Where |
|-------|-------|
| PHP bootstrap, validation, pricing and summary filters | `modules/custom-fields/class-custom-fields.php` |
| Base class every field type extends | `modules/custom-fields/class-field-base.php` (`Field_Base`) |
| Registry, shared settings, rendering, choice helpers | `modules/custom-fields/class-fields.php` (`Fields`) |
| One class per field type | `modules/custom-fields/fields/class-field-<type>.php` |
| Frontend runtime (unbundled script modules) | `modules/custom-fields/assets/frontend/js/*.js` |
| Templates | `templates/layer/layer-field.php` (shared wrapper) + `templates/layer/layer-<type>.php` (one per type) |

Unlike Conditional Logic, this module ships **no editor bundle** — there's no `assets/editor/` directory. Its layer types are registered into the *existing* "Model Structure" layer list and its own settings section into the *existing* layer-settings panel, both via plain schema arrays (`layer_types()` / `layer_settings_controls()` on `Module_Base`) rather than custom Vue UI. See [`extend-modules.md`](extend-modules.md) for what each of those overrides does.

The frontend is hand-written native ES modules, not a Vite build, so devtools shows the real file and line on a customer's site. They are listed with their imports in `Custom_Fields::frontend_script_modules()` and loaded with `wp_enqueue_script_module()`; add an `import` to a file and that list changes with it.

---

## Field Types

One class each under `fields/`, registered by `Custom_Fields::get_field_types()` in the order they appear in the editor's add-layer bar:

| Slug | Class | Stores | Notes |
|------|-------|--------|-------|
| `textfield` | `Field_Textfield` | single string | Debounced text entry (`text_entry`) |
| `textarea` | `Field_Textarea` (extends `Field_Textfield`) | single string | Debounced (`text_entry`), multi-line (`multiline`) |
| `select` | `Field_Select` (extends `Field_Choice`) | single string | Dropdown (`has_options`); shows an empty leading row as its "nothing chosen" placeholder |
| `radio` | `Field_Radio` (extends `Field_Choice`) | single string | Row of choices (`has_options`, `grouped`) |
| `checkbox` | `Field_Checkbox` (extends `Field_Choice`) | string[] | Row of choices (`has_options`, `grouped`), the only type whose value is a list (`multiple`) |
| `switch` | `Field_Switch` | boolean | Stores `true` and nothing else (`boolean`) — turning it off deactivates the layer entirely rather than storing `false` |
| `range` | `Field_Range` | number | A number on a scale (`numeric`); see [`range.md`](range.md) |
| `file-upload` | `Field_File` | string[] | Files the shopper uploaded (`file`, `multiple`): a list of references issued by `File_Uploads`; see [File Uploads](#file-uploads) |

Every value is written under the same fixed key, `Fields::VALUE_KEY` (`'value'`) — `store.customData[uid].value` — regardless of type, so consumers (Conditional Logic's `field_value` rule, for instance) can read a field without knowing which type produced it.

### Capability flags, not a type switch

Each type declares **capability flags** in `get_capabilities()` (`placeholder`, `text_entry`, `multiline`, `has_options`, `multiple`, `grouped`, `boolean`, `numeric`, `file`; the list with meanings is in `Field_Base::get_capabilities()`). The shared editor settings and the frontend's fallback handling are gated on those flags rather than on slugs. `Fields::get_slugs_with( $capability )` and its named wrappers (`get_option_slugs()`, `get_numeric_slugs()` and so on) derive the lists, and `Fields::is_numeric_type( $type )` and its siblings test one layer. A new type that declares the same flags as an existing one gets the same controls.

### What a type overrides

`Field_Base` is the contract. Only the three abstract methods are required; everything else defaults to plain-text behaviour.

| Method | Default | Overridden by |
|---|---|---|
| `get_slug()`, `get_name()`, `get_icon()` | abstract | every type |
| `get_value_label()` | `get_name()` | every bundled type |
| `get_capabilities()` | none | every bundled type |
| `resolve_value( $stored, $settings )` | stored value, else `field_default` | choice types (marked choices), switch (`field_checked`), range (normalised), file (only references that check out) |
| `get_template_args( $field, $skin )` | unchanged | choice types (options, chosen), switch and range (their handlers, the scale), file (the files and the picker's limits) |
| `get_amounts( $settings, $raw )` | the layer's own price pair | choice types (adds chosen choices), range (tiers / per unit), file (Price Per File for each file) |
| `get_summary( $raw, $settings )` | the value on one line | choice types (labels), switch (empty), range (value and unit), file (file names) |
| `is_answered( $raw, $settings )` | a non-empty value | file (at least one file that checks out) |
| `get_settings_controls()` | none | file (formats, size and count; see below) |

`get_slug()` must not translate: `Fields::register()` reads it on `after_setup_theme`, before `init`.

### Adding a field type

1. Write a `Field_Base` subclass (or extend `Field_Textfield` / `Field_Choice` if it is close to one) and register it from the `wp3dconf/custom_fields/register_types` action. The bundled types register on `after_setup_theme` (priority 20) and the action fires straight after, so it works from any plugin, whatever the load order, and from a theme's `functions.php`. It only fires when the Custom Fields module is on. Keep the class in its own file and require it inside the hook: declaring it at the top level of your plugin file is a fatal error whenever your plugin loads first, or the module is off.

   ```php
   // my-addon.php
   add_action( 'wp3dconf/custom_fields/register_types', function () {
   	require_once __DIR__ . '/class-my-date-field.php';

   	\WP3DCONF\CustomFields\Fields::register( new My_Date_Field() );
   } );
   ```

   ```php
   // class-my-date-field.php
   use WP3DCONF\CustomFields\Field_Base;

   class My_Date_Field extends Field_Base {
   	public function get_slug() { return 'my_date'; }
   	public function get_name() { return esc_html__( 'Date', 'my-addon' ); }
   	public function get_icon() { return 'calendar'; }
   	protected function get_capabilities() {
   		return array( 'placeholder' => true, 'text_entry' => true );
   	}
   	public function get_settings_controls() {
   		return array(
   			'my_date_min' => array(
   				'name' => esc_html__( 'Earliest date', 'my-addon' ),
   				'type' => 'text',
   			),
   		);
   	}
   }
   ```

   A type's own settings are added after the shared ones and shown only for that type unless they set `conditions` themselves. Registering an existing slug replaces that type.

2. Add `templates/layer/layer-<slug>.php` with the control markup. `Fields::render()` no-ops silently if this file doesn't exist — nothing crashes, the field just renders nothing.

3. Frontend: nothing, if the type behaves like the bundled type sharing its flags. The registry falls back by capability (`numeric` → range, `boolean` → switch, `has_options` → select or checkbox, anything else → text). Only a type with behaviour of its own needs a handler; see [Frontend Runtime](#frontend-runtime).

Nothing in `Custom_Fields` or `Fields` changes.

### Registry helpers

| Method | Returns |
|---|---|
| `Fields::register( Field_Base $type )` | Registers a type and hooks its markup. A slug registered twice replaces the earlier type |
| `Fields::get( $slug )` | The registered `Field_Base`, or `null` for a slug this module doesn't own |
| `Fields::supports( $slug, $capability )` | Whether a layer type carries a capability; `false` for an unknown slug |
| `Fields::is_field_type( $slug )` | Whether the slug belongs to this module |
| `Fields::get_slugs()` / `get_slugs_with( $capability )` | Registered slugs, in add-layer bar order |
| `Fields::store_call( $skin, $method, $uid, $args )` | An Alpine expression calling a store method for a layer, such as `$store.x.handleFieldSwitch( "uid", $event.target.checked )`, for a type's `get_template_args()`. Empty without a skin |
| `$type->supports( $capability )` / `$type->get_supported_capabilities()` | One capability, or all of them, after `wp3dconf/custom_fields/capabilities` |
| `$type->to_definition()` | The type as the editor sees it, after `wp3dconf/custom_fields/type_definition` |

These read the registry, which is filled on `after_setup_theme`; before that it is empty.

---

## Editor Settings

`layer_settings_controls()` adds a "Field Settings" section, gated to only appear when a custom-field layer type is selected (`Fields::gate( Fields::get_slugs() )` — a `conditions.terms` schema the layer-settings panel already understands from other layer-type-conditional controls). Individual controls within the section are further gated to their own relevant types:

| Control | Shown for |
|---------|-----------|
| `field_options` | Types with `has_options` (select, radio, checkbox) |
| `field_placeholder` | Types with `placeholder` |
| `field_default` | Types with `text_entry` or `numeric` (choice types mark defaults on the choices themselves instead) |
| `field_maxlength` | Types with `text_entry` |
| `field_checked` | Types with `boolean` (switch) |
| `field_rows` | Types with `multiline` (default: 4) |
| `field_required` | Every field type except `numeric` ones — a Range always holds a value |
| `range_*` | Types with `numeric` — the scale, input style, unit and price tiers. See [range.md](range.md). |
| `file_formats`, `file_max_size`, `file_max_files` | File Upload only, declared by `Field_File::get_settings_controls()`. Defaults: JPG, PNG, WEBP and PDF; 5 MB; 1 file |
| `file_unit_price` | Types with `file`, in the Price section: Price Per File |

Price, sale price, label and CSS class aren't repeated here — the core layer-settings controls already apply to any non-group layer.

---

## Frontend Runtime

The runtime mirrors the PHP: one file per field type, behind a registry, and one file per concern that calls the type's handler instead of testing the type. It reads its config from `window.WP3dConf.customFields` (the module's `frontend_payload()`).

| File | Role |
|---|---|
| `main.js` | Entry: imports the type files, then starts `init.js` |
| `config.js` | The payload from PHP, read on every call |
| `registry.js` | `registerType()`, `getType()`, the `BASE` handler |
| `type-text.js` / `type-choice.js` / `type-switch.js` / `type-range.js` / `type-file.js` | One handler per type, mirroring `Field_Textfield`, `Field_Choice`, `Field_Switch`, `Field_Range`, `Field_File` |
| `commit.js` | Store a value, or take the layer out when it is empty |
| `store.js` | The store methods every type shares, then each handler's own |
| `defaults.js`, `summary.js`, `pricing.js`, `required.js` | One host hook each (below) |
| `init.js` | Wiring |

`main.js` starts at once rather than on `DOMContentLoaded`, unlike the other unbundled modules. The host fires `storeInitialized` and `data/initialized` inside `Alpine.start()`, so a later listener would miss both. It subscribes on `wp3dconf/frontend:ready`, and binds straight to the store if that already exists.

### Handlers

A handler overrides only what differs from `BASE` in `registry.js`:

| Method | Purpose |
|---|---|
| `toStored( layer, raw )` | The value to store for raw control input |
| `isEmpty( value )` | An empty value takes the layer out of the configuration |
| `initialValue( layer, { active, restored } )` | The starting value, or `undefined` to leave the layer alone |
| `normalize( layer, raw, stored )` | What `store.normalizeFieldValue()` returns |
| `summary( layer, raw, label, data )` | The summary popup entry, `null` to drop it |
| `thumbnail( layer, raw )` | An image URL for the summary popup's thumbnail, `null` for none, or `undefined` (the default) to leave it to other modules |
| `thumbnailText( layer, raw )` | Short text for that box when there is no image, such as a file type; same `null` / `undefined` rules |
| `amounts( layer, raw, amounts )` | The layer's price pair |
| `isAnswered( layer, raw )` | Whether a required field counts as filled in |
| `bindStore( store )` | Add the store methods the type's markup calls |

A type registered in PHP with no handler borrows the bundled one matching its capabilities (a `file` type borrows File Upload's). A type with behaviour of its own registers a handler from its own script module. Handlers are looked up when used, so registering any time before the customer touches the field is early enough:

```js
import { registerType } from '@wp3dconf/custom-fields/registry';

registerType( 'my_date', {
	toStored( layer, raw ) {
		return /^\d{4}-\d{2}-\d{2}$/.test( raw ) ? raw : '';
	},
} );
```

### Committing a value

```js
store.handleFieldInput(uid, value);   // every type's x-on:input / x-on:change; debounced 300ms for `typedTypes`
store.handleFieldSwitch(uid, checked); // switch (type-switch.js)
store.handleFieldToggle(uid, value, checked); // one checkbox row toggling on/off (type-choice.js)
store.handleRangeInput(uid, value);   // range while dragging, debounced 120ms (type-range.js)
store.handleRangeChange(uid, value);  // range released or box left (type-range.js)
```

Debouncing is per-uid (`debounceByKey`), so typing in one field never resets another field's pending commit. Only types in `typedTypes` (i.e. `text_entry`) are debounced at all — a dropdown or radio commits the instant it changes, since waiting 300ms to reprice a discrete choice just looks broken.

An **empty** commit — an empty string, empty list, or an off switch — deactivates the layer via `store.deactivateLayer()`. A Range never deactivates; its commits are snapped onto its scale (see [range.md](range.md)). A commit to a layer that is already active fires `wp3dconf/frontend/layer/customDataUpdated`. A non-empty commit on an inactive layer calls `store.activateLayer()` directly rather than `store.handleOptionClick()`, because that helper deselects sibling options on a single-select group — which typing into a field must never trigger.

### Defaults on load

`applyDefaults()` (`defaults.js`) runs on `wp3dconf/frontend/data/initialized`, after `activeLayers` has been seeded from any restored or shareable-link state, and asks each handler's `initialValue()`:

- `switch` — activates if `field_checked` is set.
- Choice types — activates whichever options are marked `selected` in `field_options` (first one only, for non-`multiple` types).
- `range` — always holds a value, so it is seeded even when restored, clamped to its scale.
- Text types — activates with `field_default` if non-empty.

Except for the range, a restored value wins over a configured default.

### Hooks it uses

| Hook | Type | Purpose |
|------|------|---------|
| `wp3dconf/frontend/storeInitialized` | action | Binds `handleFieldInput` etc. onto the store |
| `wp3dconf/frontend/data/initialized` | action | Applies configured defaults |
| `wp3dconf/frontend/summary/layerCustomData` | filter | Relabels a field's raw `{ value: "Jane" }` entry as `{ "Text": "Jane" }` in the summary popup, using each type's `value_label`; returns `null` for a switch (its layer name already says everything) |
| `wp3dconf/frontend/price/layer_amounts` | filter | The handler's `amounts()`: chosen choices' prices, or a range's tier / per-unit price |
| `wp3dconf/frontend/submit/blocked` | filter | Blocks add-to-cart / Get a Quote submission while a required field is empty; writes the message into `store.notices[NOTICE_KEY]`, rendered by `render_required_notice()` (PHP, below) |

A required field that's currently hidden or disabled (e.g. by a Conditional Logic action) is skipped in the required check — demanding a value for a field the customer can't see or interact with would deadlock the form.

---

## Required-Field Validation (PHP)

The frontend blocks submission client-side, but two server-side gates exist for anything that bypasses the page (a stale cached form, a hand-built request):

| Gate | Hooked on | Function |
|------|-----------|----------|
| Get a Quote | `wp3dconf/quote/validation_errors` | `validate_required_fields()` — adds a `WP_Error` |
| WooCommerce cart | `woocommerce_add_to_cart_validation` | `validate_cart_required_fields()` — calls `wc_add_notice()` and returns `false` |

Both funnel into `missing_required_fields()` → the static, DB-free `missing_from_layers( $layers, $custom_data, $unavailable )`, which is the actual decision logic, kept separate from the WordPress/WooCommerce plumbing around it so it can be reasoned about (and tested) without a database.

`$unavailable` lists the layers hidden or disabled when the form was sent: the forms post `store.getUnavailableLayers()` as `wp3dconf-unavailable-layers`, read back by `Configurator_Utils::unavailable_layers()`. A required field that Conditional Logic hid therefore doesn't block the order on the server either. The list only relaxes the required check; it never changes a price.

The "you missed one" message itself is printed by `render_required_notice()`, hooked on `wp3dconf/frontend/form/hidden_fields` — it's placed there (inside the form, alongside the hidden fields) rather than beside the field itself, because the field can be behind the summary/form overlay when the customer tries to submit.

---

## Pricing

A priced field (or a priced choice within one) needs to show correctly in the live total *and* bill correctly on the quote, PDF and WooCommerce cart — three separate code paths. Two filters keep them in sync, mirroring each other on the JS and PHP sides:

- **PHP:** `wp3dconf/data/custom_data_layer_price`, hooked as `field_layer_price()`, which returns the type's `get_amounts()`. `Configurator_Utils::get_custom_data_price()` calls this filter and defaults to zero for *any* layer with custom data — without this hook, a priced field would show its price in the live JS total but bill as free everywhere server-rendered (quote email, PDF invoice, WooCommerce order).
- **JS:** `wp3dconf/frontend/price/layer_amounts`, hooked as `fieldAmounts()` in `pricing.js`, which returns the handler's `amounts()`.

For a choice type both add the layer's own configured `price`/`sale_price` plus the price of every currently-selected choice (a checkbox can have several, so this is a sum, not a single lookup). A range adds its tier or per-unit price; see [`range.md`](range.md).

---

## File Uploads

`Field_File` (`file-upload`) stores what `File_Uploads` (`class-file-uploads.php`) hands back. Uploads go to their own endpoint as soon as the shopper picks a file, so the quote and cart forms only ever post references, never file data.

### The stored value

`customData[uid].value` is a list of references, `<attachment id>:<secret>`. The secret goes only to the browser that uploaded the file; the attachment keeps an HMAC of it under `_wp3dconf_field_file`, with the configurator, the field uid, the original name, the extension and the size. `File_Uploads::get_files( $raw, $settings )` turns a value back into files, keeping only references that check out, in a format the field allows, without duplicates, up to Max Files. Price, summary and the required check all go through it, so a guessed or copied attachment id counts for nothing.

```php
foreach ( File_Uploads::get_files( $raw, $settings ) as $file ) {
	// $file: ref, id, name, ext, size, url, config_id, uid.
}
```

### Endpoints

Both are open to logged-out visitors and check the `wp3dconf_nonce` nonce.

| AJAX action | POST fields | Does |
|---|---|---|
| `wp3dconf_upload_field_file` | `nonce`, `config_id`, `uid`, `file` (multipart) | Stores one file for a File Upload field on a published configurator (or a draft the user can edit). Returns `ref`, `id`, `name`, `ext`, `size` and `url` |
| `wp3dconf_remove_field_file` | `nonce`, `ref` | Deletes the file, only with its full reference and only while no order or quote has used it. Answers the same either way |

A visitor may upload 60 files an hour across all fields (`file_upload_rate_limit`).

### What is checked

1. The field exists, is a `file` type, and the configurator is published.
2. PHP reported no upload error, and the size is under the field's Max File Size, which is capped at `wp_max_upload_size()`.
3. The extension is one the field allows. `jpeg` counts as `jpg`, and `tiff` as `tif`.
4. The content passes the format's check (`File_Uploads::passes_check()`), then `file_valid`.

The file is then stored as an attachment under `uploads/wp3dconf-files/<year>/<month>/` with a random name and the extension from the allowlist, so it cannot be found by guessing and is never run as code. Each folder gets an empty `index.php`. Images get their size and EXIF read, but no resized copies. WordPress's own type check is skipped, because it misreads GLB, STL and AI files; the content checks below replace it.

### Formats

| Extension | Group | Content check |
|---|---|---|
| `jpg`, `png`, `webp`, `gif` | image | `getimagesize()` reads it, and its real type matches the extension |
| `pdf` | document | `%PDF-` in the first 1024 bytes |
| `docx`, `xlsx`, `pptx` | document | A zip with `[Content_Types].xml` and the part that kind of file has (`word/document.xml` and so on), and no `vbaProject` or `activeX` parts |
| `odt`, `ods` | document | A zip whose `mimetype` entry names that kind, with no `Basic/` or `Scripts/` macros |
| `txt`, `csv` | document | No NUL bytes, and no `<?php`, `<script`, `<html`, `<svg`, `<iframe` or `<object` in the first 8 KB |
| `tif` | print | The TIFF byte-order header, `II*` or `MM*` |
| `psd` | print | `8BPS` |
| `heic` | print | An `ftyp` box with an HEIF brand |
| `ai` | print | A PDF or PostScript header |
| `eps` | print | `%!PS`, or the DOS EPS header |
| `glb` | model | `glTF`, version 2, and the length in its header matches the file |
| `stl` | model | Binary, whose triangle count matches the file length, or text starting with `solid` |
| `obj` | model | Text with at least one `v`, `vt`, `vn` or `f` line |

The macro checks need PHP's zip extension; without it only the zip signature is checked.

`File_Uploads::BLOCKED_EXTENSIONS` are never accepted, whatever `file_formats` returns: SVG, HTML and XML (they run script when opened from the site's domain), PHP and other scripts or programs, macro-enabled Office files (`docm`, `xlsm`, `pptm` and their templates), and archives (`zip`, `rar`, `7z`, `gz`, `tar`), whose contents cannot be checked. `gltf` is not offered, because it points at separate `.bin` and texture files.

To add a format, give it a `check`: a built-in name, or a callable taking the file path.

```php
add_filter( 'wp3dconf/custom_fields/file_formats', function ( $formats ) {
	$formats['dxf'] = array(
		'label' => 'DXF',
		'mime'  => 'image/vnd.dxf',
		'group' => 'model',
		'check' => function ( $path ) {
			$head = (string) file_get_contents( $path, false, null, 0, 4096 );

			return false === strpos( $head, "\0" ) && false !== strpos( $head, 'SECTION' );
		},
	);

	return $formats;
} );
```

### In the browser

`layer-file-upload.php` draws the same dashed drop zone as the Custom Image Upload trigger: a round icon, a label, a "click, or drag and drop" hint and a line with the formats and limits. The real `<input type="file">` is hidden visually but stays focusable, so the label opens it and the keyboard reaches it. Files already attached are listed above the picker, each with a thumbnail (images) or its extension, its size, and a remove button. The picker hides once Max Files is reached.

`type-file.js` does the rest:

- `initFieldFiles( uid, el )`, from the wrapper's `x-init`, reads `data-files` (the files `File_Uploads::get_files()` vouched for) and `data-limits`, drops any restored reference the server did not render, and makes the picker a drop zone. Its listeners are removed if the field is set up again.
- `handleFieldFiles( uid, event )` and drops call `addFieldFiles( uid, files )`. It refuses a wrong format, a file over Max File Size, and files beyond Max Files without uploading them, then sends each file to `wp3dconf_upload_field_file` with `XMLHttpRequest`, so the list shows progress.
- A finished upload's reference is committed with the field's other files, in list order. `removeFieldFile( uid, key )` commits the rest and asks `wp3dconf_remove_field_file` to delete the file. Removing the last file takes the layer out.
- `fieldFileList()`, `fieldFileError()` and `canAddFieldFile()` are what the template draws from. State lives on the store as `store.fieldFiles[ uid ]`.

Only a reference whose details the page knows counts towards the live price, the summary popup and the required check: one the server just returned, or one the template rendered. That is the same set `File_Uploads::get_files()` counts in PHP, so the price shown is the price charged. The summary popup shows file names, never the references, which hold each file's secret, and the first uploaded image as the row's thumbnail, the way Custom Image Upload shows its image. A field with no image shows its first file's type instead, such as PDF, as text in the same grey box, through the host's `wp3dconf/frontend/summary/layerThumbnailText` filter. `summary.js` passes it to the host's `wp3dconf/frontend/summary/layerThumbnail` filter, which the popup draws as a CSS background; only an http(s) URL is used, with `(`, `)` and quotes escaped.

### After the upload

`File_Usage` (`class-file-usage.php`) follows a file from upload to order:

| When | What happens |
|---|---|
| Uploaded | Marked `_wp3dconf_field_file_pending` with the time |
| Added to a WooCommerce cart (`woocommerce_add_cart_item_data`) | The pending time restarts, so a shopper who comes back to the cart still has the file |
| A quote is saved (`save_post_wp3dconf-quotes`, used by Get a Quote and Contact Form 7) | The mark is cleared and `_wp3dconf_field_file_used` gets a `quote:<id>` row |
| A WooCommerce order line is created (`woocommerce_checkout_create_order_line_item`, classic and block checkout) | The mark is cleared and an `order:<id>` row is added |
| Still pending after 7 days (`file_retention_days`) | Deleted by the daily `wp3dconf_sweep_field_files` cron, 200 per run. A run that hits the limit schedules another an hour later |

A file is only marked used, or linked, for the field it was uploaded to, on the configurator it was uploaded to (`File_Usage::files_for_field()`). A reference copied into another field is ignored there.

Every server-rendered summary (the cart, the order, the quote screen and email, the PDF invoice) prints the file names from `Field_File::get_summary()`, then a "View file" link per file from `wp3dconf/data/custom_data_html`. The links point at the attachment URL, which is public but unguessable. Change them with `file_links`.

### Not done yet

Uploads are not part of WordPress's personal data export and erase tools. The cron event is not cleared when the plugin is deactivated, which is harmless, as it has no callback then. Custom Image Upload's own uploads (`_wp3dconf_temp_image`) are not swept, because nothing records which of them an order used.

---

## Rendering

`Fields::render()` is hooked once per type on `wp3dconf/frontend/controls/layer_<type>_html`, by `Fields::register()`. It resolves everything a template needs — labels, values, Alpine handler expressions, price HTML — into one `$field` array via `prepare_field()`, so a `layer-<type>.php` template only has to lay out its own control. Every key is present for every type with a neutral default, then the type fills in its own through `get_template_args()`, so a theme's template never meets a missing key:

```php
// templates/layer/layer-textfield.php (abbreviated)
<input
  type="text"
  id="<?php echo esc_attr( $field['field_id'] ); ?>"
  value="<?php echo esc_attr( $field['value'] ); ?>"
  <?php echo $field['required'] ? 'required aria-required="true"' : ''; ?>
  x-on:input="<?php echo esc_attr( $field['input_handler'] ); ?>"
/>
```

`layer-field.php` is the shared wrapper — it prints the label and price, then delegates to `layer-<type>.php` for the actual control. Override `layer-field.php` to change the frame around every field type at once, or an individual `layer-<type>.php` to change one type's control.

### `wp_kses()` allowance

Control-item HTML is escaped with `Utils::allowed_tags()`, which by default strips most attributes off `input` and doesn't know about `textarea`, `select`, or `option` at all in a control-item context. `Fields::allowed_html_tags()`, hooked on `wp3dconf/utils/allowed_html_tags`, widens exactly what's needed (`placeholder`, `value`, `maxlength`, `required`, `checked`, plus the `textarea`/`select`/`option` tags themselves) — and only widens, so it's a no-op anywhere `input` isn't already in the allowed-tags list.

---

## Extending

**A field's value in Conditional Logic:** because every field type writes to the same `customData[uid].value` key, Conditional Logic's `field_value` rule (see [`conditional-logic.md`](conditional-logic.md)) works against any custom field automatically — there's nothing field-type-specific to wire up on that side. (`text_length` is defined but not yet wired.)

**Reading a field's value from your own addon:** `store.customData[uid].value` on the frontend; `$data->get_custom_data()[$uid][Fields::VALUE_KEY]` (or the `$custom_data` array already passed into most `Data_Repository`-adjacent filters) on the PHP side. Use `Fields::is_field_type( $layer['type'] )` to check whether a given layer is one of this module's types before assuming the shape.

---

## Extension points

Every hook below is new in 1.4.0. Filters return their first argument; return it unchanged for anything you don't mean to touch. Each per-field hook is passed the `Field_Base` type, so `$field_type->get_slug()` or `$field_type->supports( 'has_options' )` narrows a callback to some fields.

**Keep the pairs in step.** The live total, the summary popup and the required check run in the browser; the quote, PDF, cart and the server's required check run in PHP. A hook marked with a pair below changes only its own side, so change the other too, or the shopper sees one figure and is charged another.

### PHP filters

All names start with `wp3dconf/custom_fields/`.

| Filter | Args | Use it to | JS pair |
|---|---|---|---|
| `field_types` | `$types` (keyed by slug) | Remove, replace or reorder the bundled types before they register | — |
| `capabilities` | `$capabilities`, `$type` | Give a type a capability, or take one away. Read once per type, after `init` | — |
| `type_definition` | `$definition`, `$type` | Change a type's editor name, icon or summary label | — |
| `settings_controls` | `$controls` | Add, remove or change a setting in the Field Settings section | — |
| `resolved_value` | `$resolved`, `$stored`, `$settings`, `$type`, `$layer` | Change the value a control first shows | `initialValue` |
| `template_args` | `$field`, `$layer`, `$skin`, `$type` | Change what the templates receive. Change values; don't remove keys | — |
| `option_label` | `$label`, `$option`, `$price` | Change a dropdown row's text (plain text) | — |
| `field_price` | `$amounts`, `$raw`, `$settings`, `$type`, `$layer` | Change a field's price as the quote, PDF and cart charge it | `price` |
| `summary` | `$summary`, `$raw`, `$settings`, `$type`, `$uid` | Change how a value reads in the quote, email, PDF and cart | `summary` |
| `is_answered` | `$answered`, `$value`, `$settings`, `$type`, `$layer` | Decide whether a required field counts as filled in | `isAnswered` |
| `missing_fields` | `$missing`, `$layers`, `$values`, `$unavailable` | Change the required fields reported as empty | `missing` |
| `frontend_payload` | `$payload`, `$post_id` | Change `window.WP3dConf.customFields` | — |
| `editor_payload` | `$payload`, `$post_id` | Change `editorData.customFields` | — |
| `file_formats` | `$formats` (keyed by extension) | Add, remove or relabel the formats a File Upload field can allow. Blocked extensions are removed afterwards. Read once per request | — |
| `file_valid` | `$valid`, `$tmp`, `$ext`, `$settings` | Decide whether an uploaded file's content matches its format, after the built-in check | — |
| `file_upload_rate_limit` | `$limit` | Change how many files one visitor may upload per hour (default 60). Zero or less turns it off | — |
| `file_links` | `$links`, `$files`, `$uid` | Change the file links in the cart, order, quote email and PDF. Returned as-is, so escape what you add | — |
| `file_retention_days` | `$days` | Change how long a file no order or quote used is kept (default 7). Zero or less stops the sweep | — |

### PHP actions

| Action | Args | Fires |
|---|---|---|
| `wp3dconf/custom_fields/register_types` | — | Once the bundled types are registered, on `after_setup_theme`. Call `Fields::register()` here |
| `wp3dconf/custom_fields/type_registered` | `$type`, `$slug` | After each type registers, or replaces one |
| `wp3dconf/custom_fields/before_field` | `$field`, `$layer`, `$skin` | Before a field's markup. Fired in PHP, so themes that override the templates still get it |
| `wp3dconf/custom_fields/after_field` | `$field`, `$layer`, `$skin` | After a field's markup |
| `wp3dconf/custom_fields/file_uploaded` | `$response`, `$config_id`, `$uid` | After a shopper's file is stored. `$response` is what the browser receives |
| `wp3dconf/custom_fields/files_used` | `$ids`, `$used_by`, `$config_id` | After a quote or order marks its files as used. `$used_by` is such as `quote:45` or `order:120` |

Deliberately not filterable: a Range's normalised value, and the normalised choice list. The browser computes both from the layer settings itself, so a PHP-only filter would show the shopper a value or a choice the browser prices differently. Change the layer settings instead, or use `field_price` with `price`.

### JS hooks

All names start with `wp3dconf/frontend/customFields/`. Register them inside the `wp3dconf/frontend:ready` event (see [hooks-frontend.md](hooks-frontend.md)), or from a script module that loads before the configurator.

| Hook | Type | Args | Use it to | PHP pair |
|---|---|---|---|---|
| `value` | filter | `value`, `{ uid, layer, raw, store }` | Change a value just before it is stored. An empty result takes the layer out | — (the server stores what is posted) |
| `committed` | action | `{ uid, layer, value, store }` | React to a stored value; `value` is `null` when the layer was taken out | — |
| `initialValue` | filter | `next`, `{ uid, layer, active, restored, store }` | Change a field's starting value; `undefined` leaves the layer alone | `resolved_value` |
| `summary` | filter | `entry`, `{ uid, layer, raw, label, data, store }` | Change the summary popup entry; `null` drops it | `summary` |
| `thumbnail` | filter | `thumbnail`, `{ uid, layer, raw, store }` | Change the image beside a field in the summary popup; `null` for none | — |
| `thumbnailText` | filter | `text`, `{ uid, layer, raw, store }` | Change the text shown in that box when there is no image; `null` for none | — |
| `price` | filter | `amounts`, `{ uid, layer, raw, store }` | Change a field's live `{ price, salePrice }` | `field_price` |
| `isAnswered` | filter | `answered`, `{ uid, layer, raw, store }` | Decide whether a required field counts as filled in | `is_answered` |
| `missing` | filter | `missing`, `{ store }` | Change the list that blocks the submit | `missing_fields` |
| `storeBound` | action | `{ store }` | Add store methods of your own, or wrap the field ones | — |
| `fileUploaded` | action | `{ uid, layer, file, store }` | React to a file stored on a File Upload field; `file` is the server's response | `file_uploaded` |
| `removeFileOnServer` | filter | `remove`, `{ uid, ref, store }` | Keep a removed file on the server. The server only deletes a file no order or quote has used, whatever this returns | — |

For behaviour a hook can't reach, register a type handler (see [Handlers](#handlers)).

### Recipes

**Drop a bundled type.**

```php
add_filter( 'wp3dconf/custom_fields/field_types', function ( $types ) {
	unset( $types['radio'] );
	return $types;
} );
```

A configurator that already has a Radio layer keeps it saved, but the layer no longer renders.

**Charge per character of a text field, on both sides.**

```php
add_filter( 'wp3dconf/custom_fields/field_price', function ( $amounts, $raw, $settings, $field_type ) {
	if ( $field_type->supports( 'text_entry' ) ) {
		$amounts['price'] += 0.5 * mb_strlen( (string) $raw );
	}
	return $amounts;
}, 10, 4 );
```

```js
window.addEventListener( 'wp3dconf/frontend:ready', () => {
	const textTypes = window.WP3dConf.customFields?.typedTypes ?? [];

	window.WP3dConf.addFilter( 'wp3dconf/frontend/customFields/price', ( amounts, { layer, raw } ) => (
		textTypes.includes( layer.type )
			? { ...amounts, price: amounts.price + 0.5 * String( raw ?? '' ).length }
			: amounts
	) );
} );
```

**Store one field's text in capitals.** The box still shows what was typed; the summary, quote and cart read the stored value.

```js
window.addEventListener( 'wp3dconf/frontend:ready', () => {
	window.WP3dConf.addFilter( 'wp3dconf/frontend/customFields/value', ( value, { layer } ) => (
		'Engraving' === layer.name && typeof value === 'string' ? value.toUpperCase() : value
	) );
} );
```

**Add a line under every field.**

```php
add_action( 'wp3dconf/custom_fields/after_field', function ( $field ) {
	if ( $field['required'] ) {
		echo '<p class="my-required-note">' . esc_html__( 'Required', 'my-addon' ) . '</p>';
	}
} );
```

**Give your own type a shared setting.** Shared settings follow capabilities, so declaring one shows its setting. Here a `my_date` type gains Placeholder:

```php
add_filter( 'wp3dconf/custom_fields/capabilities', function ( $capabilities, $field_type ) {
	if ( 'my_date' === $field_type->get_slug() ) {
		$capabilities['placeholder'] = true;
	}
	return $capabilities;
}, 10, 2 );
```
