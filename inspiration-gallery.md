# Inspiration Gallery

**Audience:** shop owners creating inspirations in the editor, and developers styling, overriding or extending the Inspiration Gallery module.
**Applies to:** plugin v1.4.0
**Last updated:** 2026-11-11

The Inspiration Gallery module (`inspiration-gallery`, shown as **Inspiration Gallery** on wp3dconf → Modules) gives a configurator a set of ready-made looks. Each inspiration has a name, a thumbnail and a choice of options. They show as a row of thumbnails under the 3D model, under a heading set in **Settings > Text Strings**, and visitors apply a look with one click, then change it further or undo it.

It's a self-contained module built on the `Module_Base` contract described in [`extend-modules.md`](extend-modules.md).

> **Pro only.** The module is listed in `build-scripts/build-lite.php`'s strip list and required from `pro/class-wp3dconf.php`, so it is not in the Lite package.

---

## Overview

| Piece | Where |
|-------|-------|
| PHP bootstrap, validation, payloads, import remap | `modules/inspiration-gallery/class-inspiration-gallery.php` |
| Editor dialog (Vue) | `common/_dev/modules/inspiration-gallery/` → built to `modules/inspiration-gallery/assets/editor/` |
| Frontend markup (theme-overridable) | `templates/inspiration-gallery.php` |
| Frontend Alpine component | `modules/inspiration-gallery/assets/frontend/js/` (hand-written ES modules, not bundled) |
| Frontend styling | `modules/inspiration-gallery/assets/frontend/inspiration-gallery.css` |
| Applying a configuration | `applyConfiguration()` on the host store, `common/_dev/frontend/stores/index.js` |

---

## Using it (shop owner)

1. Enable **Inspiration Gallery** on **wp3dconf → Modules**.
2. Open a configurator in the editor and click the **Inspirations** (photo library) icon in the toolbar. A dialog opens.
3. Click **Add Inspiration**. The new inspiration starts with the configurator's default choices.
4. For each inspiration:
   - **Name** — shown under the thumbnail on the frontend.
   - **Thumbnail** — click the thumbnail (it has a **+** on its corner) to pick an image from the Media Library; **Remove** under it clears it. To get a picture of the model, configure the look on the product page, use the toolbar's **Download** button and upload that image.
   - **Selections** — one row per group (sub groups show their path, such as *Body › Colour*). Pick the option the group should have; a group that allows several choices takes several. **A group left empty keeps its default choice.** **Reset to defaults** sets every row back.
   - Arrows reorder inspirations; **Delete** removes the selected one.
5. Click **Save**, or use the editor's normal Save or autosave. The inspirations go with the same save request.

The heading above the thumbnails is set once for the site, under **wp3dconf → Settings → Text Strings → Inspiration Heading**, for example *High selling configurations*. It defaults to *Get inspired*.

To hide the gallery on one page, add `show_inspiration="false"` to the shortcode, for example `[wp3dconf id="12" show_inspiration="false"]`, or switch off **Show inspiration gallery** in the block or page builder settings. See [`embed-configurator.md`](embed-configurator.md).

Up to 50 inspirations per configurator.

---

## How it behaves on the frontend

- The gallery sits directly under the canvas, in the canvas column, in every skin: the heading, then a row of thumbnail cards. It shows only when the configurator has inspirations and the page has not hidden it with `show_inspiration`, and only once the configurator is ready, so it never flashes on load or shows without JavaScript.
- The row scrolls sideways when the cards do not fit. On devices with a mouse, round ‹ › arrows at the ends scroll it most of a width at a time and show only while there is more on that side; touch screens swipe and show no arrows.
- Each card shows the inspiration's thumbnail, name and total price, in the shop's currency, with the regular price struck through when it is on sale. The price is what the configuration would cost once applied (defaults filled in, missing options dropped), worked out with `store.priceConfiguration()` without applying anything, and again whenever the configurator's price changes.
- Clicking a card applies that look straight away and outlines the card. *"Name applied"* with an **Undo** link appears next to the heading.
- **Undo** goes back to what the visitor had before their first inspiration, even after trying several.
- As soon as the visitor picks an option themselves, the outline and Undo go away, so Undo never throws away their own changes.

### What "apply" does

Applying goes through `store.applyConfiguration()`, which changes the selection the same way clicks would:

- Everything leaving is deactivated first, then everything new is activated, so a swapped mesh is put back before its replacement goes on.
- Prices, canvas changes, and the `layer/activated` / `layer/deactivated` hooks run as for a click, so [Conditional Logic](conditional-logic.md) re-evaluates as usual.
- A group the inspiration does not list gets its default. A listed group gets exactly the listed options that still exist; a single-choice group keeps only one; a required group that would end up empty falls back to its default.
- `user/option_clicked` does **not** fire, so modules that react only to a visitor's own clicks (such as [Model Animation](model-animation.md)) are not triggered.

---

## Data

### Stored shape

Post meta `_wp3dconf_inspirations`:

```php
array(
  'version' => 1,
  'items'   => array(
    array(
      'id'            => 'insp_ab12cd',
      'name'          => 'Midnight Edition',
      'thumbnail_id'  => 123,
      'active_layers' => array( 'groupUid' => array( 'optionUid' ) ),
    ),
  ),
)
```

`active_layers` is the same `{ parentUid: [childUid] }` shape the frontend store and shareable links use.

The heading is the `wp3dconf_inspiration_heading_text` option, read with `Inspiration_Gallery::get_heading()`.

### Validation

`Inspiration_Gallery::normalize()` runs on save and again whenever the data is read, so a configurator that arrived through an import is cleaned the same way:

- An item with a missing or duplicate `id` is dropped. Names are sanitized, capped at 100 characters, and fall back to *Untitled*.
- A thumbnail that is not an image attachment becomes `0`; the item is kept without one.
- A selection is kept only while the option is still a child of that group in the current model structure, so deleting an option in the editor quietly removes it from every inspiration.
- A single-choice group keeps only the last option.

### Payloads

| Where | Key | Contents |
|---|---|---|
| Editor | `window.wp3dconfEditorData.inspirationGallery` | `iconId`, `items` (stored shape plus a `thumbnail_url` for the preview), `i18n` |
| Frontend | `window.WP3dConf.inspirationGallery` | `heading`, and `items`: `{ id, name, thumbnail, activeLayers }`. `thumbnail` is the `medium` size URL. Not printed when there are no inspirations or the gallery is switched off. |

### Export, import, duplicate and templates

- On save, each thumbnail's URL is added to the configurator's `_wp3dconf_attachment_urls`, the list of images an export carries.
- On import, and when creating a configurator from a template, those images are sideloaded like any other. The module then swaps each `thumbnail_id` for the new attachment through the `wp3dconf/import/remap_attachments` filter. A thumbnail that could not be sideloaded is cleared rather than left pointing at an unrelated image on the new site. This runs even when the module is disabled on that site.
- **Duplicate** copies the meta on the same site, so thumbnails stay as they are.

---

## Customising

### Template

Copy `templates/inspiration-gallery.php` to `your-theme/wp3dconf/<plugin-slug>/inspiration-gallery.php`. Keep:

- `data-wp3dconf-inspiration`, `hidden` and `x-ignore` on the root, and its `x-data="wp3dconfInspirationGallery"`;
- `x-ref="list"` and `x-on:scroll.passive="updateScroll()"` on the row, if you keep the arrows;
- `x-ref="undoText"` and its `data-template` on the Undo text;
- the price markup inside each card (`wp3dconf-inspiration-price` and its `x-bind:hidden` / `x-text` calls), or leave it out to show no prices;
- `data-id` on each card.

The template receives `$args['heading']` (plain text, may be empty), `$args['items']` (the frontend items above), `$args['config_id']` and `$args['skin']`. It is printed inside the canvas wrapper, after the canvas, on `wp3dconf/frontend/canvas/after_canvas`.

### Styling

The cards, Undo and the arrows are `<button>`s for keyboard and screen reader use, but every theme style on them (background, border, padding, hover and focus) is reset with `all: unset` in every state, and only the gallery's own look is drawn back. The only state style is a focus ring for keyboard users (`:focus-visible`), which a mouse click or tap never shows. Colours and the card width are CSS variables on `.wp3dconf-inspiration`:

```css
.wp3dconf-inspiration {
  --wp3dconf-inspiration-bg: #fff;
  --wp3dconf-inspiration-text: #1e1e1e;
  --wp3dconf-inspiration-muted: #6b6b6b;
  --wp3dconf-inspiration-accent: #1e1e1e;
  --wp3dconf-inspiration-card-width: 132px; /* 112px on screens 600px wide or narrower */
}
```

With `prefers-reduced-motion`, the arrows jump instead of scrolling smoothly.

### Applying a configuration from your own code

`applyConfiguration()` is on the host store, so any script can use it:

```js
const store = Alpine.store( `wp3dconf_${ WP3dConf.postId }` );

// Returns the configuration it replaced.
const previous = store.applyConfiguration( { activeLayers: { groupUid: [ 'optionUid' ] } } );

// Later: put it back.
store.applyConfiguration( previous );
```

`store.getConfiguration()` returns a copy of the current selection in the same shape, and `store.priceConfiguration( { activeLayers } )` returns `{ total, regularPrice }` for a configuration without applying it. See [`hooks-frontend.md`](hooks-frontend.md) for `wp3dconf/frontend/configuration/applied`.

### How the frontend starts

The frontend files are native ES modules loaded with `wp_enqueue_script_module()`, which run after the host has started Alpine. The component therefore can't be found on Alpine's first pass, so the root is printed with `x-ignore`. `mountGallery()` in `gallery.js` registers the component, removes the ignore and starts just that element with `Alpine.initTree()`.

---

## Hooks

Every part of the feature can be extended without editing the plugin. PHP hooks are standard WordPress hooks. JavaScript hooks use the host hook bus, `window.WP3dConf.addAction()` / `addFilter()`, in the editor and on the frontend.

### PHP filters

| Filter | Value | Extra arguments | Use it to |
|---|---|---|---|
| `wp3dconf/inspiration_gallery/item` | `array` cleaned item (`id`, `name`, `thumbnail_id`, `active_layers`) | `$raw` item as submitted or stored (unsanitized), `$parent_map` | Keep your own fields (sanitize them from `$raw`), or drop an item by returning a non-array. Runs on save and on every read. `id` cannot be changed; the core keys are restored if removed. |
| `wp3dconf/inspiration_gallery/max_items` | `int` 50 | — | Allow more or fewer inspirations per configurator. |
| `wp3dconf/inspiration_gallery/enabled` | `bool` true | `$post_id` | Hide the gallery for a configurator, a page or a visitor. A single page can also use `show_inspiration="false"`. |
| `wp3dconf/inspiration_gallery/thumbnail_size` | `string\|int[]` `'medium'` | `$post_id` | Use another image size on the frontend. |
| `wp3dconf/inspiration_gallery/frontend_item` | `array` `id`, `name`, `thumbnail`, `activeLayers` | `$item` stored item, `$post_id` | Add keys for the template and frontend hooks (a price, a badge), or leave an item out by returning a non-array. |
| `wp3dconf/inspiration_gallery/frontend_payload` | `array` `heading`, `items` | `$post_id` | Change the heading, or reorder or filter the whole list. Empty `items` hides the gallery. |
| `wp3dconf/inspiration_gallery/template_args` | `array` `skin`, `config_id`, `heading`, `items` | `$config_id` | Pass extra variables to an overridden template. |
| `wp3dconf/inspiration_gallery/editor_payload` | `array` `iconId`, `items`, `i18n` | `$post_id` | Send strings or settings to an editor add-on. |
| `wp3dconf/import/remap_attachments` | `array` imported meta | `$id_map`, `$post_id` | Remap attachment IDs kept in your own fields after an import. See [`hooks-backend.md`](hooks-backend.md). |

### PHP actions

| Action | Arguments | Fires |
|---|---|---|
| `wp3dconf/inspiration_gallery/saved` | `$post_id`, `$inspirations` | After the inspirations were saved from the editor. |
| `wp3dconf/inspiration_gallery/deleted` | `$post_id` | After the last inspiration was removed. |
| `wp3dconf/inspiration_gallery/before_list` | `$items`, `$config_id` | In the gallery, under the heading and before the row. |
| `wp3dconf/inspiration_gallery/after_list` | `$items`, `$config_id` | In the gallery, after the row. |
| `wp3dconf/inspiration_gallery/card_after_name` | `$item`, `$config_id` | Inside each card, after the name and price. The card is a `<button>`: print escaped phrasing content only, such as a `<span>`. |

### Frontend (JavaScript)

Every hook gets a `context` object with `store` (the configurator's Alpine store) and `gallery`, the control API below; hooks about one inspiration also get `item` (the frontend item, including keys added in PHP).

| Hook | Type | Value / arguments | Use it to |
|---|---|---|---|
| `wp3dconf/frontend/inspiration/ready` | action | `{ gallery, store }` | Keep the API, e.g. to apply an inspiration from a link. |
| `wp3dconf/frontend/inspiration/can_apply` | filter | `true`, `{ item, store, gallery }` | Return `false` to stop an inspiration being applied. |
| `wp3dconf/frontend/inspiration/configuration` | filter | `{ activeLayers }`, `{ item, store, gallery }` | Change what is applied. Add `customData` to set a Custom Text or other layer's data. |
| `wp3dconf/frontend/inspiration/applied` | action | `{ item, configuration, previous, store, gallery }` | React to an apply, e.g. for analytics. |
| `wp3dconf/frontend/inspiration/undone` | action | `{ configuration, store, gallery }` | React to Undo. `configuration` is what was put back. |
| `wp3dconf/frontend/inspiration/undo_text` | filter | `string` e.g. *"Night applied"*, `{ item, store, gallery }` | Change the text before **Undo**, plain text, or return `false` to show no Undo. |
| `wp3dconf/frontend/inspiration/price` | filter | `{ total, regularPrice }`, `{ item, store, gallery }` | Change a card's price, or return `false` to show none. |

`store.applyConfiguration()` itself also fires `wp3dconf/frontend/configuration/applied`; see [`hooks-frontend.md`](hooks-frontend.md).

The `gallery` API:

| Member | Does |
|---|---|
| `apply( id )` | Apply an inspiration, through the same filters as a click. Returns whether it was applied. |
| `undo()` | Put back the configuration from before the first apply. |
| `activeId()` | The applied inspiration's id, or `''`. |
| `refreshPrices()` | Work the card prices out again. |
| `items` | The inspirations, a `Map` by id. |
| `root` | The gallery's root element. |

Register frontend hooks on the `wp3dconf/frontend:ready` window event, as for every other frontend hook; the gallery starts after it, on `DOMContentLoaded`, so `ready` is never missed.

```js
window.addEventListener( 'wp3dconf/frontend:ready', () => {
  const { addAction, addFilter } = window.WP3dConf;

  // Apply an inspiration from a link such as #inspiration=night.
  addAction( 'wp3dconf/frontend/inspiration/ready', ( { gallery } ) => {
    const match = window.location.hash.match( /inspiration=([\w-]+)/ );

    if ( match ) {
      gallery.apply( match[ 1 ] );
    }
  } );

  // Set a layer's custom data with the look (its shape depends on the layer type).
  addFilter( 'wp3dconf/frontend/inspiration/configuration', ( configuration, { item } ) => {
    if ( item.engraving ) {
      configuration.customData = { textLayerUid: { text: item.engraving } };
    }

    return configuration;
  } );
} );
```

### Editor (JavaScript)

| Hook | Type | Value / arguments | Use it to |
|---|---|---|---|
| `wp3dconf/editor/inspiration/fields` | filter | `[]` | Return Vue components to render under the selections. Each gets the selected inspiration as `item` (edit it in place) and the dialog's `i18n`. Read each time the dialog opens. |
| `wp3dconf/editor/inspiration/new_item` | filter | the new item | Set your fields' defaults. The `id` cannot be changed. |
| `wp3dconf/editor/inspiration/groups` | filter | `[ { uid, label, multiple, options } ]`, model structure | Offer other layers in the selection rows, or hide groups. |
| `wp3dconf/editor/inspiration/payload` | filter | `{ items }` | Change what is sent on save. Keep extra item fields on the PHP `item` filter or they are dropped. |
| `wp3dconf/editor/inspiration/item_added` / `item_removed` | action | the item | React to an inspiration being added or deleted. |
| `wp3dconf/editor/inspiration/saved` | action | the items | React to a successful save. |

### Example: a badge on each card

```php
// Keep the field on save and read.
add_filter( 'wp3dconf/inspiration_gallery/item', function ( $item, $raw ) {
	$item['badge'] = isset( $raw['badge'] ) && is_scalar( $raw['badge'] ) ? sanitize_text_field( (string) $raw['badge'] ) : '';
	return $item;
}, 10, 2 );

// Send it to the frontend.
add_filter( 'wp3dconf/inspiration_gallery/frontend_item', function ( $frontend_item, $item ) {
	$frontend_item['badge'] = $item['badge'] ?? '';
	return $frontend_item;
}, 10, 2 );

// Print it on the card.
add_action( 'wp3dconf/inspiration_gallery/card_after_name', function ( $item ) {
	if ( ! empty( $item['badge'] ) ) {
		echo '<span class="my-badge">' . esc_html( $item['badge'] ) . '</span>';
	}
} );
```

In the editor, a component returned from `wp3dconf/editor/inspiration/fields` edits `item.badge`, and it is saved with the rest.

---

## Limitations

- Inspirations cannot set custom data, such as a Custom Text layer's text or a Custom Image Upload image, and the editor only lists `option` layers. Other layers follow their group's default: one that is on by default stays on with the visitor's text or image, and one that is not is switched off.
- The thumbnail is uploaded by hand; the editor cannot capture one from the canvas yet.
- A card's price uses the visitor's current custom data and does not run conditional logic, so a layer that a rule would change once the look is applied is priced as listed.
