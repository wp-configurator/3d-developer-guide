# Conditional Logic

The Conditional Logic module lets a configurator react to the customer's own choices — show or hide a group when an option is picked, apply a discount once a total crosses a threshold, disable a layer until another one is selected. It's a self-contained module (`modules/conditional-logic/`) built entirely on the `Module_Base` contract described in [`extend-modules.md`](extend-modules.md) — it's actually that document's own worked reference implementation, so this page focuses on what the module *does* rather than how modules are wired in general.

---

## Overview

| Piece | Where |
|-------|-------|
| PHP bootstrap + save handling | `modules/conditional-logic/class-conditional-logic.php` |
| Data schema, sanitization | `modules/conditional-logic/class-data.php` |
| Editor UI (Vue) | `common/_dev/modules/conditional-logic/` → built to `modules/conditional-logic/assets/editor/` |
| Frontend evaluation engine | `common/_dev/modules/conditional-logic/frontend/` → built to `modules/conditional-logic/assets/frontend/` |

The module is toggled on **wp3dconf → Modules** like any other module, and self-registers via `Module_Base::register()` (Option B in extend-modules.md) rather than the static `class-modules.php` array — see that file's `require_once` in `includes/class-wp3dconf.php`'s `load_common()` if you need to trace how it loads before it's known to be enabled.

---

## Data Model

Everything lives under a single post meta entry, `_wp3dconf_conditional_logic` (`Data::META_KEY`), as one schema-versioned payload:

```json
{
  "version": 1,
  "groups": [
    { "uid": "g1", "title": "Discounts", "collapsed": false, "disabled": false, "order": 0 }
  ],
  "items": [
    {
      "uid": "c1",
      "title": "Show trim when body selected",
      "group_uid": "g1",
      "order": 0,
      "operator": "and",
      "disabled": false,
      "allow_reversed": true,
      "break_chaining": false,
      "logic":  [ { "uid": "l1", "rule": { "type": "is_state", "state": "selected", "target": ["body-red"], "match": "any" } } ],
      "action": [ { "uid": "a1", "rule": { "type": "set_state", "state": "show", "target": ["trim-group"] } } ]
    }
  ]
}
```

- **Groups** are purely organisational — a way to fold related conditions together in the editor and disable them as a set (`disabled` on a group short-circuits every item in it). They carry no logic of their own.
- **Items** are the actual conditions: an `operator`-joined list of `logic` clauses (the IF side) and a list of `action` clauses (the THEN side). `allow_reversed` means the action's inverse is also applied when the condition stops being true (e.g. a `show` action's reverse is `hide`); `break_chaining` stops downstream conditions from re-evaluating in the same pass once this one fires (see [Evaluation Order](#evaluation-order)).
- Every clause (`logic[]` / `action[]` entry) wraps one **rule**, discriminated on `rule.type`.

**Greenfield versioning policy:** `Data::get( $post_id )` discards and resets to `Data::defaults()` if the stored `version` doesn't match `Data::VERSION` — there is no migration path between schema versions. If you bump `Data::VERSION`, every existing configurator's conditional logic is silently wiped on next read; plan a real migration in `Data::get()` before doing that in production.

### Rule Types

Fourteen rule types are defined and sanitized (so third-party code or a future editor build can rely on them round-tripping), but only a subset is actually wired into the editor UI and the frontend runtime in this build — the rest are preserved on save/load but inert. The gate is `Data::SUPPORTED_LOGIC_TYPES` / `Data::SUPPORTED_ACTION_TYPES`, kept in sync with the editor's own `utils/capabilities.ts`.

| Section | Type | Supported (wired) | Shape |
|---|---|:---:|---|
| Logic (IF)  | `is_state` | ✅ | `{ state, target: uid[], match: 'any'\|'all' }` — `state` ∈ `selected`, `deselected`, `shown`, `hidden`, `enabled`, `disabled`, `opened`, `closed` |
| Logic (IF)  | `option_price` | ✅ | `{ target: uid, comparator, value }` |
| Logic (IF)  | `total_price` | ✅ | `{ comparator, value }` |
| Logic (IF)  | `layer_count` |  | `{ target: uid, comparator, value }` |
| Logic (IF)  | `text_length` |  | `{ target: uid, comparator, value }` |
| Logic (IF)  | `field_value` |  | `{ target: uid, comparator, value }` — comparator is numeric or string (`equals`, `contains`, `starts_with`, ...) |
| Logic (IF)  | `formula_result` |  | `{ target: uid, comparator, value }` |
| Action (THEN) | `set_state` | ✅ | `{ state, target: uid[] }` — `state` ∈ `select`, `deselect`, `show`, `hide`, `enable`, `disable`, `open`, `close` |
| Action (THEN) | `apply_discount` |  | `{ mode: 'percent'\|'fixed', value, scope: 'total'\|'item'\|uid }` |
| Action (THEN) | `set_limit` |  | `{ target: uid, limit: 'max_chars'\|'min_chars'\|'max_selections'\|'min_selections', value }` |
| Action (THEN) | `set_value` |  | `{ target: uid, value: string\|bool\|string[] }` |
| Action (THEN) | `set_price` |  | `{ target: uid, mode: 'fixed'\|'add'\|'subtract'\|'multiply', value }` |
| Action (THEN) | `set_required` |  | `{ target: uid, value: bool }` |
| Action (THEN) | `show_message` |  | `{ message, style: 'info'\|'warning'\|'error' }` |

`comparator` for numeric rule types is one of `equal`, `not_equal`, `greater_than`, `less_than`, `greater_or_equal`, `less_or_equal`; `field_value` additionally accepts the string comparators `equals`, `not_equals`, `contains`, `not_contains`, `starts_with`, `ends_with`.

If you're extending the runtime to support one of the not-yet-wired types, the payload is already there waiting for you — add the evaluation/application branch in the frontend runtime and flip the capability flag; no data-layer change is needed.

---

## Frontend Runtime

The frontend bundle (`modules/conditional-logic/assets/frontend/conditional-logic.min.js`) binds once per configurator instance, driven entirely by the host's existing hook bus — it adds no polling and no new DOM observers.

### Binding

```
window.WP3dConf.addAction('wp3dconf/frontend/data/initialized', ({ store }) => {
  // one evaluator instance per configurator, bound here
});
```

It waits for `wp3dconf/frontend:ready` if the hook bus isn't present yet (the host frontend bundle loads as a deferred ES module, so ordering isn't guaranteed), same handshake described in `extend-modules.md`'s frontend entry section.

### Re-evaluation triggers

Once bound, the evaluator listens on these host events and re-runs every condition whenever one fires:

- `wp3dconf/frontend/layer/activated`
- `wp3dconf/frontend/layer/deactivated`
- `wp3dconf/frontend/layer/shown`
- `wp3dconf/frontend/layer/hidden`
- `wp3dconf/frontend/layer/enabled`
- `wp3dconf/frontend/layer/disabled`
- `wp3dconf/frontend/group/opened`
- `wp3dconf/frontend/group/closed`
- `wp3dconf/frontend/data/priceUpdated`

### Evaluation order

Each pass:

1. Conditions are sorted by their group's `order`, then their own `order` within the group, then `uid` — so evaluation order matches the order items appear in the editor.
2. A `disabled` item, or one whose `group_uid` points at a disabled group, is skipped entirely.
3. Each item's `logic` clauses are evaluated and combined with its `operator` (`and`/`or`); the result is compared against whether the item fired last pass.
4. On a `false → true` transition, its `action` clauses run. On a `true → false` transition, its inverse actions run **only if** `allow_reversed` is set (state actions invert via a fixed opposite-pairs map: `select`↔`deselect`, `show`↔`hide`, `enable`↔`disable`, `open`↔`close`).
5. If an item fires and has `break_chaining` set, the pass stops — items after it in evaluation order are not evaluated this pass.
6. Because an action can itself trigger the very events being listened to (e.g. a `set_state: show` action fires `layer/shown`, which is on the trigger list), the whole pass loop re-runs until nothing changes, capped at **8 iterations** — past that it logs a convergence warning ("Check for contradictory rules") and stops, rather than looping forever on a cyclic rule set.

### `is_state` against a group

`is_state` targeting a group UID (rather than a leaf layer) doesn't check the group's own selection state — it recurses into the group's descendant leaf layers and applies `match` (`any`/`all`) across them. A group's `opened`/`closed` state is the one exception evaluated directly on the group itself, since "opened" has no meaning for a leaf.

### Runtime warnings

On bind, the evaluator walks every condition once and warns (via `WP3dConfUtils.warn`) about:

- A condition whose `logic` or `action` clauses reference only layer UIDs that no longer exist in the model (a layer deleted after the condition was written) — it will never fire, or its action will do nothing.
- A condition referencing a group with zero descendant leaf layers — that clause can never be satisfied.

These mirror the same checks surfaced as inline warnings in the editor UI (`missingLayer` / `deletedLayerTooltip` i18n strings), so a broken reference is visible both while editing and at runtime.

### Internal event bus

The evaluator emits its own events via `window.WP3dConf.doAction()`, namespaced under `wp3dconf/cl/`, separate from the host's own frontend hooks — use these if you need to observe the module's own lifecycle rather than the underlying layer changes it reacts to:

| Event | Fires when | Payload |
|-------|-----------|---------|
| `wp3dconf/cl/bound` | The evaluator has bound and run its first pass | `{ store, payload }` |
| `wp3dconf/cl/condition/applied` | A condition transitions false → true and its actions run | `{ item, store }` |
| `wp3dconf/cl/condition/reversed` | A condition transitions true → false and its inverse actions run (`allow_reversed` only) | `{ item, store }` |
| `wp3dconf/cl/action/applied` | (reserved for future per-action granularity; not currently emitted separately from `condition/applied`) | — |

```js
window.WP3dConf.addAction('wp3dconf/cl/condition/applied', function({ item, store }) {
  console.log('Condition fired:', item.title);
}, 10);
```

---

## PHP Side

### `editor_payload()` / `window.editorData.conditionalLogic`

```js
{
  iconId: 'conditional-logic',
  metaKey: '_wp3dconf_conditional_logic',
  version: 1,
  data: { version, groups, items },   // Data::get( $post_id )
  i18n: { /* every string the Vue UI needs */ },
}
```

### `frontend_payload()` / `window.WP3dConf.cl`

Kept under the short key `cl` rather than the default camelCase-slug (`conditionalLogic`) via `frontend_data_key()` — a deliberate override so the already-built frontend runtime's `window.WP3dConf.cl` read doesn't need to change if the module is ever renamed.

```js
{
  configId: 123,
  version: 1,
  data: { version, groups, items },        // Data::get( $post_id )
  layerIndex: { "body-red": ["c1"], ... },  // layer_uid -> condition_uid[], for fast lookups
}
```

`layerIndex` is built by `Data::build_layer_index()` and is not currently consumed by the shipped frontend runtime — it's there for editor-side or third-party tooling that needs "which conditions reference this layer" without walking every item.

### Saving

`save( $post_id, array $payload )` sanitizes via `Data::sanitize()` and writes to post meta, wrapped in `wp_slash()` (since `update_post_meta()` unslashes internally — without this, a backslash typed into a condition's title or a `show_message` body would be silently stripped). It rides the host's shared save pipeline (`createHostSaveSync` from `wp3dconf-module-kit`) — there's no separate AJAX endpoint.

---

## Extending

**Wiring up a not-yet-supported rule type:** the data layer already sanitizes and round-trips all fourteen types (see the [Rule Types](#rule-types) table). Wiring one up at runtime means adding an evaluation branch (for a logic type) or an application branch (for an action type) to the frontend bundle, and adding its slug to the relevant `SUPPORTED_*` array in `class-data.php` so the editor UI exposes it. There's no PHP-side gate beyond that constant — it exists purely to keep the editor from offering rule types the runtime can't yet act on.

**Reacting to conditional logic from another module or addon:** listen on the `wp3dconf/cl/*` events above rather than re-deriving state from the underlying layer events — you'll get the specific condition that fired, not just "something changed".

**Adding your own logic/action rule types entirely:** there's no filter to register a fifteenth rule type from outside the module today. If you need one, the cleanest path is forking `Data::LOGIC_RULE_TYPES` / `ACTION_RULE_TYPES` and `sanitize_rule()`'s switch — there's no plugin-side extension point for the schema itself yet.
