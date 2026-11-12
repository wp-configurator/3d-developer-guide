# Page Builder Integrations

**Audience:** developers adding support for another page builder, or changing a bundled one.
**Not for:** site owners placing a configurator (see [`embed-configurator.md`](embed-configurator.md)).
**Applies to:** plugin v1.4.0
**Last updated:** 2026-11-05

Every way of placing a configurator on a page, whether the shortcode, the block or a page builder element, is an *integration*. An integration only translates its host's settings into one call to `Renderer::render()`. Access checks, asset loading and skin handling stay in `Renderer`, so a new integration is small.

---

## Overview

| Class | Role |
|-------|------|
| `Integration_Base` | Abstract base: `get_slug()`, `register()`, optional `is_available()` |
| `Integration_Registry` | Loads the bundled integrations and those added through a filter |
| `Renderer` | The one place a configurator is rendered |

All live in `common/public/`. The bundled ones are in `common/public/integrations/`.

Bundled integrations load only when their host is active. The registry runs a host check first (for example `defined( 'ELEMENTOR_VERSION' )`) and only then requires the integration file, so an inactive builder loads no files or classes.

---

## 1. Write the Integration

```php
namespace My_Addon;

use WP3DCONF\Integration_Base;
use WP3DCONF\Renderer;

class My_Builder_Integration extends Integration_Base {

	public function get_slug() {
		return 'my-builder';
	}

	public function is_available() {
		return defined( 'MY_BUILDER_VERSION' );
	}

	public function register() {
		// Hook into the builder here. Runs once on `init` priority 9.
		add_action( 'my_builder/register_elements', array( $this, 'register_element' ) );
	}
}
```

`is_available()` defaults to `true`. `register()` is only called when it returns true.

---

## 2. Add It to the Registry

```php
add_filter( 'wp3dconf/integrations', function ( $integrations ) {
	$integrations['my-builder'] = new My_Addon\My_Builder_Integration();
	return $integrations;
} );
```

The list is keyed by slug. Unset a key to remove a bundled integration, or set the same key to replace it.

`Integration_Base` and its abstract methods are public API. They are used by code outside this repository, so do not rename them or change what they return.

---

## 3. Render

In the builder's render callback, call the renderer and output the result:

```php
echo Renderer::render( array(
	'id'       => $id,
	'skin'     => $skin,
	'settings' => Renderer::pick_settings( $skin, $values, 'setting_' ),
) );
```

The markup comes from the skin templates, which escape their own output. `Renderer::render()` returns an empty string when the configurator is unpublished, has no model, or is password-protected.

Helpers for building the builder's controls from the registered skins:

| Helper | Returns |
|--------|---------|
| `Renderer::get_configurator_options()` | Published configurators, ID => title |
| `Renderer::get_skin_options()` | Skins, name => label |
| `Renderer::get_settings_fields()` | Every skin option merged by key: `field` (schema entry) and `skins` (skins that declare it) |
| `Renderer::pick_settings( $skin, $values, $prefix )` | The settings to pass to `render()`, taken from the builder's saved values |
| `Renderer::get_placeholder( $id, $skin )` | A text box to show inside the builder's editor |

Loop over `get_settings_fields()` to create one control per option, so a new skin option shows up with no builder changes. If the builder can show a control for a given skin only, use each field's `skins` list.

Pass values on as the builder saves them. The skin sanitises them against its schema, and boolean options accept `yes`/`no`, `on`/`off`, `true`/`false` and `1`/`0`.

---

## 4. Handle the Editor and the Head

Two filters cover the cases where a builder is different from a normal page.

**Do not run the configurator inside the builder's editor.** It creates a WebGL context and is heavy and unreliable in a preview frame. Show `Renderer::get_placeholder()` there. For a builder that previews shortcodes, also return true from:

```php
add_filter( 'wp3dconf/is_editor_context', function ( $is_editor ) {
	return $is_editor || my_builder_is_editing();
} );
```

**Load assets in the head.** Scripts load at render time anyway, but the skin's critical CSS must be in the head or the configurator flashes in. The plugin already detects the shortcode and the block. If the builder stores content outside `post_content`, tell it when the page has the element:

```php
add_filter( 'wp3dconf/page_has_configurator', function ( $found ) {
	if ( $found || ! is_singular() ) {
		return $found;
	}
	return my_builder_page_uses_element( get_the_ID() );
} );
```

---

## 5. Things to Check

- **Load builder classes lazily.** A class that extends a builder's class must be required from inside `register()` (or a builder hook), never at file load, because the parent does not exist until the builder is ready.
- **One configurator per page.** The frontend bundle holds a single `WP3dConf` global and one canvas. A second `render()` call returns an editor-only notice.
- **Keep keys consistent.** Options are merged by key across skins, and the first definition wins in the builder's controls.

---

## Bundled Integrations

| Slug | File | Host check |
|------|------|------------|
| `shortcode` | `common/public/class-shortcode.php` | none |
| `gutenberg` | `integrations/class-gutenberg.php` | block API present |
| `elementor` | `integrations/class-elementor-integration.php` | `ELEMENTOR_VERSION` |
| `divi` | `integrations/class-divi-integration.php` | `ET_BUILDER_VERSION` |
| `bricks` | `integrations/class-bricks-integration.php` | `BRICKS_VERSION` |
| `beaver-builder` | `integrations/class-beaver-integration.php` | `FLBuilder` classes |
| `wpbakery` | `integrations/class-wpbakery-integration.php` | `WPB_VC_VERSION` |

The checks live in `Integration_Registry::get_bundled()`. Use the Elementor integration (`class-elementor-integration.php` and `elementor/class-elementor-widget.php`) as the reference implementation. WPBakery is the smallest, since it maps the shortcode and needs no render code.
