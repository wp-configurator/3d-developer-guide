# Creating a New Module

This document is the full process for adding a new feature module to
wp-3d-configurator, backend (PHP) and frontend (Vue/TS) together. Follow it
top to bottom for a new module; skip the parts a given module doesn't need
(not every module has a frontend runtime, for example).

The running example throughout is a hypothetical module with the slug
`my-module`. Everywhere you see `my-module` / `My_Module` / `myModule`,
substitute your own slug in the three casings.

## 0. Naming: pick a slug once, everything else follows

A module has exactly one identity: its **slug** (kebab-case, e.g.
`conditional-logic`). Every other name in the system is mechanically derived
from it - you never invent a second name:

| Thing | Derived as |
|---|---|
| PHP module directory | `modules/<slug>/` |
| Frontend/editor source directory | `common/_dev/modules/<slug>/` |
| Built editor bundle | `modules/<slug>/assets/editor/<slug>.min.{js,css}` |
| Built frontend bundle | `modules/<slug>/assets/frontend/<slug>.min.{js,css}` |
| WP script/style handles | `wp3dconf-<slug>-editor`, `wp3dconf-<slug>-frontend` |
| `editorData` / `WP3dConf` payload key | camelCase of the slug (`conditional-logic` → `conditionalLogic`) |
| `$_POST` field on save | slug with `-` → `_` (`conditional-logic` → `conditional_logic`) |
| Registry key (`Modules::$module_files`, or self-registered via `Module_Base::register()`) | the slug itself |

This is enforced by [`modules/class-module-base.php`](modules/class-module-base.php)
(`Module_Base`) on the PHP side and by
[`common/_dev/scripts/build-modules.mjs`](common/_dev/scripts/build-modules.mjs)
on the build side. As long as your directories and one PHP method
(`get_slug()`) agree on the slug, nothing else needs to be told about your
module's file paths.

## 1. Decide what your module needs

A module can supply any subset of these; nothing is mandatory beyond
`get_slug()`:

- An **editor bundle** (Vue component that plugs into the backend 3D editor) - needs `common/_dev/modules/<slug>/main.ts`
- A **frontend bundle** (runs on the public configurator page) - needs `common/_dev/modules/<slug>/frontend/main.ts`
- A **toolbar icon** in the editor
- A **custom layer type** in the "Model Structure" tab's draggable layer list
- A **global setting control** in the editor's General Settings panel (or a new section of it)
- **Data** persisted with the configurator (read in the editor, read on the frontend, written on save)
- A contribution to the **shareable-link** payload (the `?key=` flow), read back out on a later request
- Pure PHP hook logic with no JS at all (like the existing `woocommerce` module)

Whatever you don't need, you simply don't implement - `Module_Base`'s
defaults are all safe no-ops.

## 2. Scaffold the PHP side

```
modules/my-module/
├── class-my-module.php   # bootstrap + Module_Base subclass
└── class-data.php        # optional: data/sanitization layer, if you persist anything
```

### 2.1 `class-my-module.php`

```php
<?php
/**
 * My Module.
 *
 * @package wp-3d-configurator\modules\my-module\
 */

namespace WP3DCONF\My_Module;

use WP3DCONF\Module_Base;

defined( 'ABSPATH' ) || exit;

require_once WP3DCONF_PATH . 'modules/class-module-base.php';
// require_once __DIR__ . '/class-data.php'; // if you have one

class My_Module extends Module_Base {

	/**
	 * @return string
	 */
	protected function get_slug() {
		return 'my-module';
	}
}

new My_Module();
```

That's a complete, working (if inert) module: it does nothing yet, but it's
wired into both the editor and frontend hook contracts and will
auto-enqueue any bundle you add later (step 3) without touching this file
again.

### 2.2 Register it

A module needs to tell `Modules` where its bootstrap file lives. There are
two ways to do that - pick one.

**Option A - static array (default, simplest).** Add the slug + path to
`common/admin/class-modules.php`:

```php
private static $module_files = array(
	'get-a-quote'    => '/get-a-quote/class-get-a-quote.php',
	'contact-form-7' => '/contact-form-7/class-contact-form-7.php',
	'woocommerce'    => '/woocommerce/class-woocommerce.php',
	'my-module'      => '/my-module/class-my-module.php',
);
```

This is the one place a new module is *not* auto-discovered - the registry
is deliberately an explicit allow-list (see step 2.3) rather than a
directory scan, so a module can exist on disk without being wired in.

**Option B - self-registration via `Module_Base::register()`.** A module can
instead register its own path from inside its own bootstrap file, without
touching `class-modules.php` at all:

```php
// modules/my-module/class-my-module.php, near the top, after the requires
Module_Base::register( 'my-module', __FILE__ );
```

`Module_Base::register()` hooks the `wp3dconf/modules/files` filter, which
`Modules::get_module_files()` merges into the static array before deciding
what to load. `modules/conditional-logic/class-conditional-logic.php` uses
this approach - read it for the full pattern.

There's a catch worth understanding before reaching for this:
`Modules::init_modules()` (the thing that would normally `require_once` your
file only when it's enabled) can't discover a path that's only known
*inside* the very file it hasn't loaded yet. Something has to load the file
unconditionally, early enough for its top-level `Module_Base::register()`
call to run, before `Modules` is constructed. For `conditional-logic` that's
an explicit `require_once` in `includes/class-wp3dconf.php`'s
`load_common()`, placed before the `require_once` for `class-modules.php`:

```php
private function load_common() {
	// Modules.
	require_once WP3DCONF_PATH . 'modules/conditional-logic/class-conditional-logic.php';

	// Admin.
	require_once WP3DCONF_PATH . 'common/admin/class-modules.php';
	// ...
```

Because the file is now always loaded regardless of whether the module is
enabled, the enable/disable gate has to move *into* the module itself -
guard the bottom-of-file instantiation with `Modules::is_enabled()` instead
of relying on `Modules::init_modules()`'s conditional `require`:

```php
use WP3DCONF\Modules;
// ...
if ( Modules::is_enabled( 'my-module' ) ) {
	new My_Module();
}
```

The tradeoff: your class file (and anything it `require`s, like a
`class-data.php`) gets parsed on every request, even when the module is
disabled - fine for a lightweight class, worth avoiding for a module with
expensive top-level code. Option A doesn't have this cost, since
`class-modules.php`'s own loop skips the `require` entirely for a disabled
module. Reach for Option B mainly when a module needs to own its own wiring
end-to-end (e.g. it ships as a separate plugin/add-on); for a new module
bundled directly in this plugin, Option A is the simpler default.

### 2.3 Enable it

Modules are gated by the `wp3dconf_enabled_modules` option (an array of
slugs, default `['get-a-quote']`), editable from **wp3dconf → Modules**
(`common/admin/views/html-modules-page.php`) in wp-admin. A module whose
slug isn't in that option is never instantiated (see step 2.2 for exactly
where that gate lives for each registration option). During development,
either check the box in that screen or add your slug to the option
directly.

### 2.4 `Module_Base` - what you get for free, and what to override

Every hook wiring (`add_action`/`add_filter`) happens once, in
`Module_Base::__construct()`. You never call `add_action`/`add_filter`
yourself for the standard contract - you override the protected method that
backs it:

| Override this protected method... | ...to control this |
|---|---|
| `get_slug()` **(required)** | Everything in the table in step 0 |
| `init()` | Any extra hook wiring beyond the standard contract (e.g. WooCommerce-specific hooks) |
| `toolbar_icon()` | Return `['id' => ..., 'icon' => ..., 'label' => ...]` to add an editor toolbar icon, or `null` (default) for none |
| `layer_type()` | Return `['slug' => ..., 'name' => ..., 'icon' => ...]` to add a custom layer type to the "Model Structure" tab's draggable layer list, or `null` (default) for none |
| `global_settings_controls( $settings )` | Mutate the editor's global-settings schema (`Editor_Data::get_global_settings_controls()`) - e.g. add an option to an existing control such as `_wp3dconf_form`, or add a whole new settings section; returns `$settings` unchanged by default |
| `editor_payload( $post_id )` | Return an array to expose as `window.editorData.<key>` in the editor, or `null` (default) for nothing |
| `frontend_payload( $post_id )` | Return an array to expose as `window.WP3dConf.<key>` on the frontend, or `null` (default) for nothing |
| `editor_data_key()` | Override the camelCase-slug default key used by `editor_payload()` |
| `frontend_data_key()` | Override the camelCase-slug default key used by `frontend_payload()` |
| `post_field()` | Override the default `$_POST` field name (`slug` with `-`→`_`) your save payload arrives under |
| `save( $post_id, array $payload )` | Persist `$payload` (already JSON-decoded from `post_field()`) - no-op by default |
| `stored_value_payload( $share_value, $hash )` | Contribute to the payload persisted for a shareable link (the `?key=` flow, see `Frontend_Ajax_Callback::store_values_to_retrieve()`); returns `$share_value` unchanged by default |
| `retrieved_value_payload( $data, $key )` | Shape that payload back out after it's read back from storage on a later request (`Data_Repository::get_default_uids()` / `get_custom_data()`); returns `$data` unchanged by default - pair this with `stored_value_payload()` if you need to decode a field you stored |

You never touch: asset enqueueing, script handles, dev-server URLs,
`filemtime()` versioning, or `type="module"` tagging for dev bundles -
`Module_Base` derives all of it from `get_slug()` and from which files
actually exist (see step 3). If a module has no editor bundle, no CSS, or no
save payload, the corresponding piece is simply skipped - you don't stub
anything out.

**Reference implementation:** read
[`modules/conditional-logic/class-conditional-logic.php`](modules/conditional-logic/class-conditional-logic.php)
top to bottom - it exercises every override in the table above except
`init()`, `layer_type()`, `global_settings_controls()`, the shareable-link
pair, and the `*_data_key()` overrides (it overrides `frontend_data_key()`
only, to keep a short `cl` key on the wire). It's also the reference for
Option B self-registration from step 2.2: the `Module_Base::register()`
call near its top, the matching `require_once` in
`includes/class-wp3dconf.php`, and the `Modules::is_enabled()` guard around
its final `new Conditional_Logic()`. For the settings-controls convention,
see `global_settings_controls()` in
[`modules/woocommerce/class-woocommerce.php`](modules/woocommerce/class-woocommerce.php),
[`modules/get-a-quote/class-get-a-quote.php`](modules/get-a-quote/class-get-a-quote.php),
or [`modules/contact-form-7/class-contact-form-7.php`](modules/contact-form-7/class-contact-form-7.php).

### 2.5 The hook contract `Module_Base` plugs into

These are host-defined hooks (in `common/admin/class-editor.php`,
`common/admin/class-editor-data.php`, `common/admin/class-ajax-callback.php`,
`common/includes/class-data-repository.php`, and
`common/public/class-frontend.php`) that `Module_Base` already subscribes
to. You don't fire these yourself - they're listed here so you know where
your overrides actually run:

- `wp3dconf/editor/toolbar_icons` (filter) - collects every module's `toolbar_icon()`
- `wp3dconf/editor/layer_types` (filter) - collects every module's `layer_type()`
- `wp3dconf/editor/global_settings_controls` (filter) - collects every module's `global_settings_controls()`
- `wp3dconf/editor/after_enqueue` (action) - fires while `wp3dconf-editor` is enqueued; triggers your editor bundle enqueue
- `wp3dconf/editor/data` (filter) - builds `window.editorData`; collects every module's `editor_payload()`
- `wp3dconf/editor/module_script_handles` (filter, dev-mode only) - tags dev bundles `type="module"`
- `wp3dconf/editor/save_configurator` (action) - fires **after** the host has already verified nonce, post ID, and edit capability; your `save()` runs here - no separate `wp_ajax_*` handler needed
- `wp3dconf/frontend/after_enqueue` (action) - triggers your frontend bundle enqueue
- `wp3dconf/frontend/data` (filter) - builds the inline `window.WP3dConf` global; collects every module's `frontend_payload()`
- `wp3dconf/frontend/module_script_handles` (filter, dev-mode only) - same as the editor one, for the frontend dev bundle
- `wp3dconf/frontend/stored_value` (filter) - fires when a shareable link's payload is written to `wp3dconf_stored_value_<hash>`; collects every module's `stored_value_payload()`
- `wp3dconf/frontend/retrieved_value` (filter) - fires when that payload is read back out for a request carrying `?key=<hash>`; collects every module's `retrieved_value_payload()`
- `wp3dconf/modules/files` (filter) - merges any `Module_Base::register()` entries into `Modules::$module_files`; consulted once by `Modules::get_module_files()` before `init_modules()` decides what to `require`

### 2.6 If your module persists data

Keep sanitization out of the `Module_Base` subclass, in its own static
`Data` class (see `modules/conditional-logic/class-data.php` for the shape:
`meta_key()`, `defaults()`, `get( $post_id )`, `save( $post_id, $data )`,
`sanitize()`). `save()` in your `Module_Base` subclass should be a thin
call into it:

```php
protected function save( $post_id, array $payload ) {
	Data::save( $post_id, $payload );
}
```

Fire your own `do_action( 'wp3dconf/my-module/before_save', ... )` /
`after_save` inside `Data::save()` if other code (or a future module)
might need to react to your writes - that's a convention, not something
`Module_Base` provides.

## 3. Scaffold the frontend/editor source

Everything under `common/_dev/modules/<slug>/` is picked up automatically
by [`build-modules.mjs`](common/_dev/scripts/build-modules.mjs) - there is
**no per-module entry in any build config**. The build script just looks
for two possible entry files per module directory:

```
common/_dev/modules/my-module/
├── main.ts              # editor entry - presence alone enables the editor build
├── MyModuleEditor.vue    # (your component; name is up to you)
└── frontend/
    └── main.ts           # frontend entry - presence alone enables the frontend build
```

Add only the entry file(s) you need. If `main.ts` doesn't exist, no editor
bundle is built and `Module_Base::enqueue_editor_assets()` never enqueues
anything for the editor side - same for `frontend/main.ts`. This is what
"no hardcoding - if the file exists it's enqueued" means in practice: the
build script's file-existence check and the PHP base class's
file-existence check are the same convention applied on both sides of the
build.

### 3.1 Editor entry (`main.ts`)

Register your component into the host editor's own Vue app instead of
mounting a second `createApp` - the host loops over registered components
and renders them with `<component :is="component">`:

```ts
// common/_dev/modules/my-module/main.ts
import MyModuleEditor from './MyModuleEditor.vue';

window.wp3dConf?.registerComponent?.(MyModuleEditor);
```

Inside `MyModuleEditor.vue`, read your PHP-side `editor_payload()` off
`window.editorData.<editor_data_key()>` (camelCase slug by default, e.g.
`window.editorData.myModule`).

**Shared deps:** `vue` and `pinia` are marked external by
`build-modules.mjs` and resolved at runtime to the host's own
`vendor-shared.min.js` - don't bundle your own copies, just `import` them
normally; the build config handles pointing the bare specifier at the
shared singleton so your module's `pinia` store and the host's share the
same instance.

**Saving from the editor:** don't make your own AJAX request. Import
`createHostSaveSync` from `wp3dconf-module-kit`
(see [`common/_dev/src/module-kit.ts`](common/_dev/src/module-kit.ts)) and
ride the host's existing save pipeline - the main Save button, autosave,
and your own explicit save all end up carrying your payload on the same
request:

```ts
import { createHostSaveSync } from 'wp3dconf-module-kit';

const sync = createHostSaveSync({
  fieldName: 'my_module',        // matches Module_Base::post_field() default
  buildPayload: () => store.payload,
  onSaved: () => store.markSaved(),
  onSaveFailed: (message) => store.markSaveFailed(message),
});

sync.bindHostSave();   // call once, e.g. on component mount
// sync.triggerSave(); // call to explicitly ask the host to save
```

`fieldName` here **must** match what your PHP `post_field()` expects
(the default is your slug with `-` → `_`) - that's the one place the slug
convention crosses from PHP into TS by string, not by shared code, so keep
them in sync if you ever override `post_field()`.

Look at
[`common/_dev/modules/conditional-logic/composables/useConditionalLogicSave.ts`](common/_dev/modules/conditional-logic/composables/useConditionalLogicSave.ts)
for a complete example of wrapping this in a composable.

### 3.2 Frontend entry (`frontend/main.ts`)

This runs on the public page. The host frontend bundle loads deferred as an
ES module, so its hook bus (`window.WP3dConf`) may not exist yet when your
IIFE runs - wait for the `wp3dconf/frontend:ready` window event unless the
bus is already there:

```ts
// common/_dev/modules/my-module/frontend/main.ts
(function () {
  const data = window.WP3dConf?.myModule; // frontend_data_key() default
  if (!data) return;

  function bind(): void {
    // ... your runtime logic, reading `data`
  }

  if (window.WP3dConf && typeof window.WP3dConf.addAction === 'function') {
    bind();
  } else {
    window.addEventListener('wp3dconf/frontend:ready', bind, { once: true });
  }
})();
```

See
[`common/_dev/modules/conditional-logic/frontend/main.ts`](common/_dev/modules/conditional-logic/frontend/main.ts)
for the full pattern, including schema-version guarding.

This bundle is built as a standalone IIFE (not ESM) in production - don't
`import` anything that isn't either bundled directly or safe to duplicate
per-module; there's no shared-vendor mechanism on the frontend side the way
there is for the editor's `vue`/`pinia`.

### 3.3 TypeScript project config

Copy `common/_dev/modules/conditional-logic/tsconfig.json` and
`shims-vue.d.ts` into your module directory unchanged (they just extend the
root TS config so `vue-tsc --build` picks your module up as part of
`pnpm run build:editor`'s type-check step).

## 4. Build

```
cd common/_dev
pnpm install        # first time only
pnpm run build       # full production build, all modules included
```

`pnpm run build` runs, in order: `build:editor` (host editor bundle +
type-check) → `build:vendor-shared` → `build:module-kit` → `build:frontend`
(host frontend bundle) → `build:modules` (every module's editor/frontend
bundle, via `build-modules.mjs`). The order matters - `vendor-shared` and
`module-kit` share an output target with the host editor build and would be
wiped if built first.

Module output lands at:
```
modules/my-module/assets/editor/my-module.min.{js,css}
modules/my-module/assets/frontend/my-module.min.js
```
exactly the paths `Module_Base` looks for. Nothing to configure - adding a
new module directory under `common/_dev/modules/` is enough for
`build-modules.mjs` to pick it up on the next `pnpm run build:modules` (or
full `build`).

### 4.1 Dev mode (HMR)

```
pnpm run dev:backend    # editor dev server, port 3000
pnpm run dev:frontend   # frontend dev server, port 3001
```

With `WP3DCONF_DEV_MODE` true (the plugin-wide dev flag, currently hardcoded
in `includes/class-wp3dconf.php`), `Module_Base` skips the built-asset
lookup entirely and instead checks whether
`common/_dev/modules/<slug>/main.ts` / `frontend/main.ts` exist on disk; if
so it points the browser at `http://localhost:3000/modules/<slug>/main.ts`
/ `http://localhost:3001/modules/<slug>/frontend/main.ts` respectively -
the same convention, just checking the source file instead of the built
one. No built assets are required to be present while developing.

## Reference: complete existing module

[`modules/conditional-logic/`](modules/conditional-logic/) (PHP) and
[`common/_dev/modules/conditional-logic/`](common/_dev/modules/conditional-logic/)
(TS/Vue) together are a complete, non-trivial example exercising every piece
described above: editor UI, frontend runtime, toolbar icon, data
persistence, host-save-pipeline integration, self-registration via
`Module_Base::register()` (step 2.2, Option B), and a custom (non-default)
frontend data key. It does not exercise `layer_type()`,
`global_settings_controls()`, or the shareable-link pair - for those, see
the module references called out in section 2.4.
