# Custom Fields

The Custom Fields module adds layer types that ask the customer for a value — engraving text, a note, a name, a choice from a list — rather than just letting them pick between pre-built visual options. A field's value rides through the same `customData[ uid ]` pipeline every other layer already uses, so it shows up in the summary, the quote/cart submission, the PDF invoice and the shareable link for free. This module adds **no storage of its own** and nothing to import/export: a field is a layer, and its configuration is layer settings.

---

## Overview

| Piece | Where |
|-------|-------|
| PHP bootstrap + validation | `modules/custom-fields/class-custom-fields.php` |
| Field type table + rendering | `modules/custom-fields/class-fields.php` |
| Frontend runtime (hand-authored, not built) | `modules/custom-fields/assets/frontend/custom-fields.js` |
| Templates | `templates/layer/layer-field.php` (shared wrapper) + `templates/layer/layer-<type>.php` (one per type) |

Unlike Conditional Logic, this module ships **no editor bundle** — there's no `assets/editor/` directory. Its six layer types are registered into the *existing* "Model Structure" layer list and its own settings section into the *existing* layer-settings panel, both via plain schema arrays (`layer_types()` / `layer_settings_controls()` on `Module_Base`) rather than custom Vue UI. See [`extend-modules.md`](extend-modules.md) for what each of those overrides does.

The frontend script is also notable for *not* being a Vite build output: `Module_Base::enqueue_frontend_assets()` falls back to `<slug>.js` when a module ships no built `<slug>.min.js`, and this module deliberately uses that fallback (see the file's own header comment).

---

## Field Types

Defined in `Fields::get_types()`, in the order they appear in the editor's add-layer bar:

| Slug | Name | Stores | Notes |
|------|------|--------|-------|
| `textfield` | Text Field | single string | Debounced text entry (`text_entry`) |
| `textarea` | Text Area | single string | Debounced (`text_entry`), multi-line (`multiline`) |
| `select` | Select | single string | Dropdown (`has_options`); shows an empty leading row as its "nothing chosen" placeholder |
| `radio` | Radio | single string | Row of choices (`has_options`) |
| `checkbox` | Checkbox | string[] | Row of choices (`has_options`), the only type whose value is a list (`multiple`) |
| `switch` | Switch | boolean | Stores `true` and nothing else (`boolean`) — turning it off deactivates the layer entirely rather than storing `false` |

Every value is written under the same fixed key, `Fields::VALUE_KEY` (`'value'`) — `store.customData[uid].value` — regardless of type, so consumers (Conditional Logic's `field_value` / `text_length` rules, for instance) can read a field without knowing which type produced it.

### Capability flags, not a type switch

Settings and frontend behaviour are gated on **capability flags** on each type's definition (`placeholder`, `text_entry`, `multiline`, `has_options`, `multiple`, `boolean`) rather than on a hardcoded list of slugs. `Fields::get_slugs_with( $capability )` and its named wrappers (`get_option_slugs()`, `get_multiline_slugs()`, `get_boolean_slugs()`, `get_text_entry_slugs()`) derive everything from those flags. This is deliberate: a future field type inherits the right editor controls and frontend handling automatically just by declaring the same flags — nothing else needs to change to add, say, a `date` type that behaves like a text field.

### Adding a field type

1. Add an entry to `Fields::get_types()` with the right capability flags.
2. Add its slug to `Fields::get_slugs()` (kept as a literal array rather than derived from `get_types()`, because `get_types()`'s labels call `esc_html__()` at plugin-bootstrap time, before it's safe to load translations — see the method's own docblock).
3. Add `templates/layer/layer-<slug>.php` with the control markup. `Fields::render()` no-ops silently if this file doesn't exist — nothing crashes, the field just renders nothing.

Nothing in `Custom_Fields` (the module bootstrap class) needs to change — `render()`, the settings controls, the pricing filter, and the frontend runtime are all type-driven off `Fields::get_types()` and the capability flags.

---

## Editor Settings

`layer_settings_controls()` adds a "Field Settings" section, gated to only appear when a custom-field layer type is selected (`Fields::gate( Fields::get_slugs() )` — a `conditions.terms` schema the layer-settings panel already understands from other layer-type-conditional controls). Individual controls within the section are further gated to their own relevant types:

| Control | Shown for |
|---------|-----------|
| `field_options` | Types with `has_options` (select, radio, checkbox) |
| `field_placeholder` | Types with `placeholder` |
| `field_default` | Types with `text_entry` (choice types mark defaults on the choices themselves instead) |
| `field_maxlength` | Types with `text_entry` |
| `field_checked` | Types with `boolean` (switch) |
| `field_rows` | Types with `multiline` (default: 4) |
| `field_required` | Every field type |

Price, sale price, label and CSS class aren't repeated here — the core layer-settings controls already apply to any non-group layer.

---

## Frontend Runtime

`custom-fields.js` teaches the host's Alpine store one new method, `handleFieldInput()`, that every field template's `x-on:input`/`x-on:change` calls, plus `handleFieldSwitch()` and `handleFieldToggle()` for the boolean and multi-select cases respectively. It reads its config from `window.WP3dConf.customFields` (the module's `frontend_payload()`), and is entirely type-agnostic — a new field type needs nothing added here as long as it's declared with the right capability flags in the PHP type table.

### Committing a value

```js
store.handleFieldInput(uid, value);   // textfield / textarea / select — debounced 300ms if `typedTypes` includes the layer's type
store.handleFieldSwitch(uid, checked); // switch
store.handleFieldToggle(uid, value, checked); // one checkbox row toggling on/off
```

Debouncing is per-uid (`debounceByKey`), so typing in one field never resets another field's pending commit. Only types in `typedTypes` (i.e. `text_entry`) are debounced at all — a dropdown or radio commits the instant it changes, since waiting 300ms to reprice a discrete choice just looks broken.

An **empty** commit — an empty string, empty list, or an off switch — deactivates the layer via `store.deactivateLayer()`. A non-empty commit on an inactive layer calls `store.activateLayer()` directly rather than `store.handleOptionClick()`, because that helper deselects sibling options on a single-select group — which typing into a field must never trigger.

### Defaults on load

`applyDefaults()` runs on `wp3dconf/frontend/data/initialized` (after `activeLayers` has already been seeded from any restored/shareable-link state, so a restored value always wins over a configured default):

- `switch` — activates if `field_checked` is set.
- Choice types — activates whichever options are marked `selected` in `field_options` (first one only, for non-`multiple` types).
- Everything else — activates with `field_default` if non-empty.

### Hooks it uses

| Hook | Type | Purpose |
|------|------|---------|
| `wp3dconf/frontend/storeInitialized` | action | Binds `handleFieldInput` etc. onto the store |
| `wp3dconf/frontend/data/initialized` | action | Applies configured defaults |
| `wp3dconf/frontend/summary/layerCustomData` | filter | Relabels a field's raw `{ value: "Jane" }` entry as `{ "Text": "Jane" }` in the summary popup, using each type's `value_label`; returns `null` for a switch (its layer name already says everything) |
| `wp3dconf/frontend/price/layer_amounts` | filter | Adds the selected choice's price to the layer's own price |
| `wp3dconf/frontend/submit/blocked` | filter | Blocks add-to-cart / Get a Quote submission while a required field is empty; writes the message into `store.notices[NOTICE_KEY]`, rendered by `render_required_notice()` (PHP, below) |

A required field that's currently hidden or disabled (e.g. by a Conditional Logic action) is skipped in the required check — demanding a value for a field the customer can't see or interact with would deadlock the form.

---

## Required-Field Validation (PHP)

The frontend blocks submission client-side, but two server-side gates exist for anything that bypasses the page (a stale cached form, a hand-built request):

| Gate | Hooked on | Function |
|------|-----------|----------|
| Get a Quote | `wp3dconf/quote/validation_errors` | `validate_required_fields()` — adds a `WP_Error` |
| WooCommerce cart | `woocommerce_add_to_cart_validation` | `validate_cart_required_fields()` — calls `wc_add_notice()` and returns `false` |

Both funnel into `missing_required_fields()` → the static, DB-free `missing_from_layers( $layers, $custom_data )`, which is the actual decision logic, kept separate from the WordPress/WooCommerce plumbing around it so it can be reasoned about (and tested) without a database.

The "you missed one" message itself is printed by `render_required_notice()`, hooked on `wp3dconf/frontend/form/hidden_fields` — it's placed there (inside the form, alongside the hidden fields) rather than beside the field itself, because the field can be behind the summary/form overlay when the customer tries to submit.

---

## Pricing

A priced field (or a priced choice within one) needs to show correctly in the live total *and* bill correctly on the quote, PDF and WooCommerce cart — three separate code paths. Two filters keep them in sync, mirroring each other on the JS and PHP sides:

- **PHP:** `wp3dconf/data/custom_data_layer_price`, hooked as `field_layer_price()`. `Configurator_Utils::get_custom_data_price()` calls this filter and defaults to zero for *any* layer with custom data — without this hook, a priced field would show its price in the live JS total but bill as free everywhere server-rendered (quote email, PDF invoice, WooCommerce order).
- **JS:** `wp3dconf/frontend/price/layer_amounts`, hooked as `addOptionPrice()` in the frontend runtime.

Both add the layer's own configured `price`/`sale_price` plus the price of every currently-selected choice (a checkbox can have several, so this is a sum, not a single lookup).

---

## Rendering

`Fields::render()` is hooked once per type on `wp3dconf/frontend/controls/layer_<type>_html` (registered in a loop in `Custom_Fields::init()`, one `add_action()` call per entry in `Fields::get_slugs()`). It resolves everything a template needs — labels, values, Alpine handler expressions, price HTML — into one `$field` array via `prepare_field()`, so a `layer-<type>.php` template only has to lay out its own control:

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

**A field's value in Conditional Logic:** because every field type writes to the same `customData[uid].value` key, Conditional Logic's `field_value` and `text_length` logic rule types (see [`conditional-logic.md`](conditional-logic.md)) work against any custom field automatically — there's nothing field-type-specific to wire up on that side.

**Reading a field's value from your own addon:** `store.customData[uid].value` on the frontend; `$data->get_custom_data()[$uid][Fields::VALUE_KEY]` (or the `$custom_data` array already passed into most `Data_Repository`-adjacent filters) on the PHP side. Use `Fields::is_field_type( $layer['type'] )` to check whether a given layer is one of this module's types before assuming the shape.
