# Troubleshooting

WP 3D Configurator provides logging tools on both the JavaScript and PHP sides to help you debug issues during development.

---

## JavaScript Logger

`WP3dConfUtils` exposes a lightweight logger that is **silent by default**. It does not output anything to the browser console unless explicitly enabled. The enabled state is stored in `localStorage` and persists across page reloads — so there is zero noise for end users who happen to open DevTools.

### Enabling & Disabling

Run these directly in your browser's developer console:

```js
// Turn logging on
WP3dConfUtils.enableLogger();

// Turn logging off
WP3dConfUtils.disableLogger();
```

### Methods

```js
const { log, warn, error, info, debug } = window.WP3dConfUtils;
```

| Method | Maps to | Description |
|--------|---------|-------------|
| `log(...args)` | `console.log` | General output |
| `warn(...args)` | `console.warn` | Non-critical warnings |
| `error(...args)` | `console.error` | Errors |
| `info(...args)` | `console.info` | Informational messages |
| `debug(...args)` | `console.debug` | Verbose debug output |

All methods accept any number of arguments, identical to their native `console` counterparts.

### Usage

```js
window.addEventListener('wp3dconf/frontend:ready', function() {
  const { addAction } = window.WP3dConf;
  const { log, warn } = window.WP3dConfUtils;

  addAction('wp3dconf/frontend/layer/activated', function({ uid, layer }) {
    log('Layer activated:', uid, layer);
  }, 10);

  addAction('wp3dconf/frontend/data/priceUpdated', function({ store }) {
    log('Price updated:', store.price);
  }, 10);
});

window.addEventListener('wp3dconf/editor:ready', function() {
  const { addAction } = window.WP3dConf;
  const { log, warn } = window.WP3dConfUtils;

  addAction('wp3dconf/editor/configurator-saved', function({ store }) {
    log('Configurator saved. Layer count:', Object.keys(store.layers).length);
  }, 10);

  addAction('wp3dconf/editor/layer-deleted', function({ layer }) {
    warn('Layer deleted — this cannot be undone:', layer);
  }, 10);
});
```

---

## PHP Logger

The plugin includes a `Logger` class for recording debug information during PHP execution. Logs are stored in `wp_options` and are viewable from the plugin's admin debug panel.

### Accessing the Logger

```php
$logger = wp3dconf()->logger;
```

### Methods

| Method | Level | Description |
|--------|-------|-------------|
| `info( $message, $context )` | `INFO` | General informational messages |
| `warning( $message, $context )` | `WARNING` | Non-critical issues worth monitoring |
| `error( $message, $context )` | `ERROR` | Failures that need attention |
| `debug( $message, $context )` | `DEBUG` | Verbose development output |

```php
$logger = wp3dconf()->logger;

$logger->info( 'Order submitted', [ 'order_id' => 123 ] );
$logger->warning( 'API rate limit approaching', [ 'requests' => 950, 'limit' => 1000 ] );
$logger->error( 'Payment failed', [ 'order_id' => 456, 'reason' => 'Card declined' ] );
$logger->debug( 'Processing step', [ 'step' => 'validation', 'order_id' => 789 ] );
```

### What Each Entry Captures

Every log entry automatically records:

| Field | Description |
|-------|-------------|
| `timestamp` | MySQL datetime at time of logging |
| `level` | `INFO`, `WARNING`, `ERROR`, or `DEBUG` |
| `message` | The message string |
| `context` | The `$context` array |
| `file` | Basename of the calling file |
| `line` | Line number of the calling code |
| `user_id` | ID of the currently logged-in WordPress user |
| `ip` | Remote IP address |

### Retrieving & Exporting Logs

```php
$logger = wp3dconf()->logger;

// Get all stored log entries
$logs = $logger->get_logs();

// Export as plain text — useful for support tickets or bug reports
$text = $logger->export_as_text();

// Clear all stored logs
$logger->clear_logs();
```

Logs are stored in `wp_options` under the key `wp3dconf_debug_logs`, newest first, capped at the last **50 entries**.

### Usage in an Addon

```php
add_action( 'wp3dconf/frontend/before_skin_display', function( $skin_instance, $post_id ) {
  wp3dconf()->logger->debug( 'Rendering skin', [
    'skin_id' => $skin_instance->get_skin_id(),
    'post_id' => $post_id,
  ] );
}, 10, 2 );
```