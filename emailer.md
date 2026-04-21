# Emailer

The plugin includes a queued email system accessible via `wp3dconf()->emailer`. Emails can be sent immediately or scheduled for future delivery, with automatic exponential-backoff retry logic and a daily cron fallback for permanently failed emails.

---

## Accessing the Emailer

```php
$emailer = wp3dconf()->emailer;
```

---

## Sending Emails

### Send Immediately

```php
wp3dconf()->emailer->send_now( $to, $subject, $message, $args );
```

| Parameter  | Type   | Description |
|------------|--------|-------------|
| `$to`      | String | Recipient email address |
| `$subject` | String | Email subject (supports mail tags) |
| `$message` | String | Email body HTML (supports mail tags and conditional blocks) |
| `$args`    | Array  | Values passed to mail tags and validation callbacks |

### Schedule for Later

```php
// Send after 1 day
wp3dconf()->emailer->send_later( $to, $subject, $message, 1, $args, 'my_email_name' );

// Send at a specific Unix timestamp
wp3dconf()->emailer->send_later( $to, $subject, $message, strtotime( '+3 days' ), $args );
```

| Parameter    | Type            | Description |
|--------------|-----------------|-------------|
| `$send_time` | int\|float      | Number of days from now, or a Unix timestamp |
| `$name`      | String          | Optional identifier used in schedule logs |

Returns the auto-increment email ID on success, or `false` on failure.

---

## Follow-up Sequences

Schedule a series of emails tied to a single sequence ID. Passing the same `$unique_id` again cancels the previous sequence before creating the new one — useful for resetting sequences on re-purchase.

```php
$sequence_id = wp3dconf()->emailer->create_followup_sequence(
  $customer_email,
  array(
    array( 'days' => 1, 'subject' => 'Thank you!',      'message' => '...' ),
    array( 'days' => 3, 'subject' => 'How is it going?', 'message' => '...' ),
    array( 'days' => 7, 'subject' => 'Reminder',         'message' => '...' ),
  ),
  array( 'customer_name' => $name, 'order_id' => $order_id ),
  $unique_id,       // Optional: pass a UID to cancel and replace an existing sequence
  'my_sequence'     // Optional: name prefix used in schedule logs
);
```

You can also specify a `timestamp` instead of `days` for individual emails in the sequence:

```php
array( 'timestamp' => strtotime( 'next Monday 9:00 AM' ), 'subject' => '...', 'message' => '...' ),
```

---

## Mail Tags

Mail tags are placeholders replaced with dynamic values at send time. Use `{tag_name}` syntax in the subject or message body.

### Built-in Tags

| Tag | Output |
|-----|--------|
| `{site_name}` | WordPress blog name |
| `{site_url}` | WordPress site URL |
| `{customer_name}` | Display name looked up from the recipient email address |
| `{notes}` | Value of `$args['notes']`, passed through `wpautop` |

### Registering a Custom Tag

```php
wp3dconf()->emailer->register_shortcode( 'order_id', function( $args ) {
  return isset( $args['order_id'] ) ? '#' . $args['order_id'] : '';
} );
```

Use it in any subject or message string:

```html
<p>Your order <strong>{order_id}</strong> has been received.</p>
```

The callback receives the full `$args` array passed to `send_now()` or `send_later()`.

---

## Conditional Blocks

Wrap content in `{% tag %}...{%}` to show it only when the named tag evaluates as truthy. Prefix with `!` to invert the condition.

```html
{% has_notes %}
  <p><strong>Notes:</strong> {notes}</p>
{%}

{% !has_notes %}
  <p>No additional notes were provided.</p>
{%}
```

Conditions are evaluated against `$args`. A value is truthy if it is a non-empty string, a number greater than zero, a non-empty array, or boolean `true`.

---

## Validation Callbacks

Pass a callable as the final argument to `send_later()` or `create_followup_sequence()` to conditionally skip sending at delivery time. The callback receives `$args` and must return `true` for the email to be sent.

> **Important:** Only string callables (function names) or `array( 'ClassName', 'method' )` static references can be used. Closures cannot be serialized and will be silently ignored.

```php
// Only send if the order is still in 'completed' status at send time
wp3dconf()->emailer->send_later(
  $customer_email,
  'Your order is ready',
  $message,
  1,
  array( 'order_id' => $order_id ),
  'order_ready',
  array( 'My_Addon_Callbacks', 'validate_order_completed' ) // validation callback
);
```

```php
class My_Addon_Callbacks {
  public static function validate_order_completed( $args ) {
    $order = wc_get_order( $args['order_id'] );
    return $order && $order->get_status() === 'completed';
  }
}
```

For sequences, a global validation callback can be passed as the last argument and applies to all emails in the sequence unless an individual email defines its own.

---

## Retry Logic

If an email fails to send, the system automatically retries using exponential backoff:

| Attempt | Delay |
|---------|-------|
| Retry 1 | 5 min |
| Retry 2 | 10 min |
| Retry 3 | 20 min |

After 3 failed attempts the email is marked `failed`. A daily cron job (runs at 2 AM) picks up failed emails from the last 15 days and reschedules them, provided they failed more than 1 hour ago.

---

## Queue Management

```php
$emailer = wp3dconf()->emailer;

// Cancel a single scheduled email by ID
$emailer->cancel_email( $id );

// Cancel all emails in a sequence
$emailer->cancel_sequence( $sequence_id );

// Manually retry a failed email immediately (resets retry count)
$emailer->manual_retry( $id );

// Get details for a specific email
$email = $emailer->get_email_by_id( $id );

// Get all emails in a sequence
$emails = $emailer->get_sequence_emails( $sequence_id );

// Get all currently pending/retrying emails
$pending = $emailer->get_pending_emails();

// Get queue statistics
$stats = $emailer->get_queue_stats();
// Returns: [ 'scheduled' => 3, 'completed' => 12, 'failed' => 1, 'total' => 16, ... ]

// Remove completed/canceled/skipped emails older than 30 days
$emailer->cleanup_old_emails( 30 );
```

### Email Statuses

| Status | Description |
|--------|-------------|
| `scheduled` | Waiting to be sent |
| `retrying` | Failed at least once; retry is scheduled |
| `completed` | Sent successfully |
| `failed` | Exhausted all retry attempts |
| `skipped` | Validation callback returned false |
| `canceled` | Manually canceled |

---

## Configuration

```php
$emailer = wp3dconf()->emailer;

// Change max retry attempts (default: 3)
$emailer->set_max_retries( 5 );

// Change base retry delay in minutes (default: 5)
$emailer->set_retry_delay( 10 );

// Change how many days back the daily cron considers failed emails (default: 15)
$emailer->set_daily_retry_threshold( 7 );

// Disable the daily retry cron entirely
$emailer->set_daily_retry( false );

// Check when the next daily retry is scheduled
$next = $emailer->get_next_daily_retry(); // Returns 'Y-m-d H:i:s' or false
```