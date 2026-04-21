# Creating a Custom Skin

Skins control how the configurator is rendered on the frontend — the layout, the controls structure, the canvas placement, and any surrounding UI. The plugin ships with two built-in skins (`default` and `accordion`). You can create your own by extending `Skin_Base` and registering it via the `wp3dconf/register_skins` hook.

---

## Overview

| Class | Role |
|-------|------|
| `Skin_Base` | Abstract base class all skins must extend |
| `Skin_Registry` | Static registry — stores and retrieves skin class references |
| `Configurator_Skin` | Factory — instantiates the correct skin for a given configurator |

---

## 1. Create the Skin Class

Your skin must extend `WP3DCONF\Skin_Base` and implement the `display()` method. Place this file inside your addon plugin.

```php
<?php

namespace My_Addon;

use WP3DCONF\Skin_Base;

defined( 'ABSPATH' ) || exit;

class My_Custom_Skin extends Skin_Base {

  /**
   * Unique skin identifier.
   * Must match the name used when registering.
   *
   * @var string
   */
  protected $skin_id = 'my-skin';

  /**
   * Render the configurator.
   *
   * @param bool $output Echo output directly.
   * @return string|void
   */
  public function display( $output = true ) {
    ob_start();
    ?>
    <div class="my-skin-wrap">
      <?php echo $this->get_canvas_html(); ?>
      <?php echo $this->get_controls_html(); ?>
      <?php echo $this->get_form_html(); ?>
    </div>
    <?php
    $html = ob_get_clean();

    if ( $output ) {
      echo $html; // phpcs:ignore WordPress.Security.EscapeOutput.OutputNotEscaped
    } else {
      return $html;
    }
  }
}
```

---

## 2. Register the Skin

Register inside the `wp3dconf/register_skins` action, which fires on `plugins_loaded` at priority 20.

```php
<?php

use WP3DCONF\Skin_Registry;
use My_Addon\My_Custom_Skin;

add_action( 'wp3dconf/register_skins', function() {
  Skin_Registry::register( 'my-skin', My_Custom_Skin::class );
} );
```

You can also register an object instance. The skin system will `clone` it and call `set_post_id()` automatically.

```php
add_action( 'wp3dconf/register_skins', function() {
  Skin_Registry::register( 'my-skin', new My_Custom_Skin() );
} );
```

---

## 3. Available HTML Helpers

`Skin_Base` provides helper methods that render template partials. Call these inside your `display()` method to compose the skin layout.

| Method | Description |
|--------|-------------|
| `get_canvas_html()` | The 3D canvas element |
| `get_controls_html()` | Full controls tree (groups, subgroups, options) |
| `get_controls_inner_html( $layers )` | Controls inner container for a specific layer set |
| `get_controls_item( $layer )` | A single control item |
| `get_total_price_html()` | Total price display element |
| `get_summary_html()` | Selection summary |
| `get_form_html()` | Add to cart / Get a Quote form |
| `get_required_fields_html()` | Required field validation messages |

All methods return HTML as a string by default. Pass `true` to echo directly.

```php
// Return as string (default)
$canvas = $this->get_canvas_html();

// Echo directly
$this->get_canvas_html( true );
```

---

## 4. Enqueuing Skin Assets

Override `register_assets()` to enqueue scripts and styles specific to your skin. This method is called once per skin — duplicate enqueuing is handled automatically by the base class.

```php
protected function register_assets() {
  wp_enqueue_style(
    'my-skin-style',
    plugin_dir_url( __FILE__ ) . 'assets/my-skin.css',
    array( 'wp3dconf-frontend' ),
    '1.0.0'
  );

  wp_enqueue_script(
    'my-skin-script',
    plugin_dir_url( __FILE__ ) . 'assets/my-skin.js',
    array( 'wp3dconf-frontend' ),
    '1.0.0',
    true
  );
}
```

### Critical CSS (Preventing FOUC)

Override `get_critical_css()` to inject minimal above-the-fold CSS directly into `<head>` before your stylesheet loads. This prevents a flash of unstyled content while assets load.

```php
protected function get_critical_css() {
  return '
    .my-skin-wrap { visibility: hidden; }
    .my-skin-wrap.is-ready { visibility: visible; }
  ';
}
```

### Inline CSS

Use `inline_css()` to inject dynamic CSS rules derived from configurator settings (colours, dimensions, etc.).

```php
public function display( $output = true ) {
  $this->inline_css( array(
    '.my-skin-wrap' => 'max-width: 1200px; margin: 0 auto;',
  ) );

  // ... rest of display()
}
```

---

## 5. Accessing Configurator Data

Use `$this->get_data()` to retrieve a `Data_Repository` instance for the current configurator post.

```php
public function display( $output = true ) {
  $data   = $this->get_data();
  $title  = $data->get_title();
  $layers = $data->get_layers();
  $style  = $data->get_style();

  // ...
}
```

### Common Data Methods

| Method | Returns |
|--------|---------|
| `get_title()` | Configurator post title |
| `get_model_url()` | GLB model URL |
| `get_base_price()` | Base price (integer) |
| `get_layers()` | Full layer tree array |
| `get_default_uids()` | Default active layer UIDs |
| `get_global_settings()` | Global configurator settings |
| `get_style()` | Style settings array |
| `get_lighting()` | Lighting settings |
| `get_camera_presets()` | Camera preset positions |
| `get_form()` | Active form type (`wc-cart-form` or `get-a-quote-form`) |
| `get_attachment_urls()` | Map of attachment IDs to URLs |

---

## 6. Accessing the Alpine.js Store

Each skin instance has a `$this->store` property — a reference string to the Alpine.js store for this configurator (e.g. `$store.wp3dconf_123`). Use it to bind PHP-rendered HTML to reactive store values.

```php
// Output a live-bound hidden field
echo '<input type="hidden" name="active_layers"
  x-bind:value=\'JSON.stringify(' . esc_attr( $this->store ) . '.activeLayers)\'>';

// Conditionally show an element based on store state
echo '<div x-show="' . esc_attr( $this->store ) . '.price > 0">
  Price: <span x-text="' . esc_attr( $this->store ) . '.formattedPrice"></span>
</div>';
```

---

## 7. Wrapper Classes

Override `get_wrapper_classes()` to add custom classes to the controls wrapper element. The base class always includes `wp3dconf-controls-wrap` and `wp3dconf-element`.

```php
public function get_wrapper_classes() {
  $classes   = parent::get_wrapper_classes();
  $classes[] = 'my-skin-controls';
  $classes[] = 'my-skin-' . $this->skin_id;
  return $classes;
}
```

---

## 8. Full Example

A complete minimal skin with custom layout, assets, and store binding.

```php
<?php

namespace My_Addon;

use WP3DCONF\Skin_Base;

defined( 'ABSPATH' ) || exit;

class My_Custom_Skin extends Skin_Base {

  protected $skin_id = 'my-skin';

  protected function get_critical_css() {
    return '.my-skin-wrap { opacity: 0; } .my-skin-wrap.wp3dconf-ready { opacity: 1; transition: opacity .3s; }';
  }

  protected function register_assets() {
    wp_enqueue_style(
      'my-skin-style',
      plugin_dir_url( __FILE__ ) . 'assets/my-skin.css',
      array( 'wp3dconf-frontend' ),
      '1.0.0'
    );
  }

  public function get_wrapper_classes() {
    $classes   = parent::get_wrapper_classes();
    $classes[] = 'my-skin-controls';
    return $classes;
  }

  public function display( $output = true ) {
    $data  = $this->get_data();
    $title = $data->get_title();

    ob_start();
    ?>
    <div class="my-skin-wrap wp3dconf-element-<?php echo esc_attr( $this->get_post_id() ); ?>">

      <h2 class="my-skin-title"><?php echo esc_html( $title ); ?></h2>

      <div class="my-skin-layout">
        <div class="my-skin-canvas">
          <?php echo $this->get_canvas_html(); ?>
        </div>

        <div class="my-skin-sidebar">
          <?php echo $this->get_controls_html(); ?>
          <?php echo $this->get_total_price_html(); ?>
          <?php echo $this->get_form_html(); ?>
        </div>
      </div>

      <?php echo $this->get_summary_html(); ?>

    </div>
    <?php
    $html = ob_get_clean();

    if ( $output ) {
      echo $html; // phpcs:ignore WordPress.Security.EscapeOutput.OutputNotEscaped
    } else {
      return $html;
    }
  }
}
```

**Registration:**

```php
<?php

use WP3DCONF\Skin_Registry;
use My_Addon\My_Custom_Skin;

add_action( 'wp3dconf/register_skins', function() {
  Skin_Registry::register( 'my-skin', My_Custom_Skin::class );
} );
```

---

## 9. Removing or Replacing a Built-in Skin

Use the `wp3dconf/registered_skins` filter to modify the registry after all skins have been registered.

```php
// Remove the built-in accordion skin
add_filter( 'wp3dconf/registered_skins', function( $skins ) {
  unset( $skins['accordion'] );
  return $skins;
} );

// Replace the default skin with your own
add_filter( 'wp3dconf/registered_skins', function( $skins ) {
  $skins['default'] = My_Addon\My_Replacement_Skin::class;
  return $skins;
} );
```