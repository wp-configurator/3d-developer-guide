# Model Animation

**Audience:** shop owners building animations in the editor, and developers integrating or extending the Model Animation module.
**Not for:** 3D artists. This covers the plugin, not authoring shape keys (see [model-animation-blender-guide.md](model-animation-blender-guide.md)).
**Applies to:** plugin v1.3.5
**Last updated:** 2026-10-16

The Model Animation module (`morph-animation`, shown as **Model Animation** on wp3dconf → Modules) moves parts of the 3D model through its **morph targets** (shape keys) — open a bag to show its interior, lift a lid, extend a handle. Animations are named, built in the editor from whatever the loaded GLB exposes, and played two ways:

- **From a group** — a group plays the animation chosen for it when a visitor opens it, and plays it back when it is closed or another group is opened.
- **From a dropdown** over the canvas — the visitor switches the model between named poses (e.g. *Closed* / *Open*).

It's a self-contained module built on the `Module_Base` contract described in [`extend-modules.md`](extend-modules.md). Nothing is tied to a specific model; an animation with no movements is simply the model's **resting pose**.

> **Pro only.** The module is listed in `build-scripts/build-lite.php`'s strip list and required from `pro/class-wp3dconf.php`, so it is not in the Lite package.

---

## Overview

| Piece | Where |
|-------|-------|
| PHP bootstrap, sanitization, payloads | `modules/morph-animation/class-morph-animation.php` |
| Editor dialog (Vue) | `common/_dev/modules/morph-animation/` → built to `modules/morph-animation/assets/editor/` |
| Shared movement editor + math | `common/_dev/src/components/Morph/MorphTracksEditor.vue`, `src/utils/morph.ts`, `src/utils/morphEvaluate.ts`, `src/stores/morphTriggers.ts` |
| Frontend runtime | `common/_dev/modules/morph-animation/frontend/` → built to `modules/morph-animation/assets/frontend/morph-animation.min.js` |
| Dropdown styling | `modules/morph-animation/assets/frontend/morph-animation.css` |

---

## Using it (shop owner)

1. Enable **Model Animation** on **wp3dconf → Modules**.
2. Load a model that has shape keys, then click the **Animations** (play) icon in the editor toolbar. A draggable dialog opens.
3. **Add Animation**. Each animation is a tab with:
   - **Name** — shown in the frontend dropdown.
   - **Play when the page loads** — marks it as the default (badge: *On load*). Only one animation can be the default.
   - **Movements** (rows) — see below.
   - **Duration** (0.05–10 s) and **Easing** (Linear, Ease in, Ease out, Ease in and out).
4. Click **Save** — or use the editor's normal Save / autosave; the animations ride the same save request.
5. To play one from a group: select a **group** or **sub group** layer, open its **Animation** settings section, and pick the animation under **Trigger**.

### Movements

Each movement drives one shape key on one part of the model:

| Field | Meaning |
|---|---|
| **Part** | The mesh, by name. |
| **Movement** | The morph target on that mesh, by name. |
| **From / To** (Amount) | Influence at the start and end of the animation, 0–1 (0 = not at all, 1 = fully). |
| **Start / End** (Timing) | The slice of the animation this movement plays over, 0–1. Use it to chain movements one after another (e.g. lid lifts during 0–0.5, flap opens during 0.5–1). |

Rules the saver enforces: a second movement for the same part + target is dropped (two writers on one influence would fight); `start`/`end` are swapped if reversed; everything is clamped to 0–1.

Movements are matched to the model by **mesh and target name, never index**, so they survive re-exporting the GLB. A movement whose part or target no longer exists is shown with a warning in the editor and skipped at runtime.

### Resting pose

An animation with **no movements** is valid and useful: activating it sends every other animation back to rest. Create one called *Closed* and use it as the default.

---

## How it behaves on the frontend

Only **one animation is active at a time** — the last one activated. The active animation plays forward; every other plays back. Interrupting an animation part-way carries on from where it is, in either direction.

### Group rules

Driven by the visitor's own clicks only (`user/group_toggled`, `user/option_clicked`). Initial state and [Conditional Logic](conditional-logic.md) open groups through other store methods and **do not** trigger animations.

| Visitor action | Result |
|---|---|
| Opens a group that has an animation | It plays — unless it's already the active one (e.g. a parent plays the same animation), then nothing restarts. |
| Opens a group with **no** animation | Model returns to rest — unless the group sits inside a group whose animation is showing; a sub group is part of its parent's view and leaves it alone. |
| Closes the group whose animation is showing | It plays back to rest — unless another open group still plays the same animation. |
| Closes any other group | Nothing. |
| Clicks an option | The animation of its nearest animated ancestor group is re-applied if something else (dropdown, another group) took over. |

### Dropdown

When at least one animation exists, a dropdown is printed over the canvas via the `wp3dconf/frontend/canvas/after_canvas` action. Nothing is printed for a configurator with no animations. It follows the animator: if a group action wins, the dropdown blanks back to its placeholder so it never claims a state that isn't applied. It's keyboard accessible (arrow keys, Home/End, Enter/Space, Escape, Tab).

### Default animation

If an animation is marked **Play when the page loads** it is activated once on bind, after the model loads, without a click. Any later action replaces it.

### Other behaviour worth knowing

- **Reduced motion:** with `prefers-reduced-motion: reduce` the pose changes instantly.
- **Background tabs:** frame time is capped at 0.1 s so a tab waking up doesn't jump to the end.
- **Swapped models:** a model brought in by a `swap_mesh` layer (`replacementModelLoaded`) is re-bound and put in the pose already showing.
- **Frustum culling is disabled** on animated meshes, since culling uses the unmorphed bounds.
- **No work when idle:** the frame loop only runs while something is moving, and a configurator with no animations never touches the hook bus.

---

## Data Model

### Animations — `_wp3dconf_morph_triggers` (post meta)

```json
{
  "default": "open",
  "items": [
    {
      "id": "closed",
      "label": "Closed",
      "animation": { "duration": 1, "easing": "easeInOut", "tracks": [] }
    },
    {
      "id": "open",
      "label": "Open",
      "animation": {
        "duration": 1.2,
        "easing": "easeOut",
        "tracks": [
          { "mesh": "Lid", "target": "Open", "from": 0, "to": 1, "start": 0, "end": 0.6 }
        ]
      }
    }
  ]
}
```

Limits: **50** animations per configurator (`MAX_TRIGGERS`), **200** movements per animation (`MAX_TRACKS`), ids ≤ 40 chars (`sanitize_key`), labels ≤ 100 chars (empty → "Untitled"). Duplicate or unusable ids are dropped; `default` falls back to `''` if it names an animation that isn't there. Saving with no valid animation deletes the meta.

### Group assignment — layer setting `morph_animation`

A group stores only the **id** of its animation inside its layer settings (`_wp3dconf_layer_settings[ <uid> ]['morph_animation']`), so it's saved, exported, imported and duplicated with the rest of the layer settings. Storing an id rather than a copy lets the frontend see that two groups share an animation and leave it running.

On every configurator save, `normalize_saved_settings()` (hooked on `wp3dconf/editor/save_configurator` at priority 20, after the module's own save) cleans each value to an existing animation id and **removes the key** when it points at nothing — so deleting an animation in the dialog un-assigns it from every group.

> The `$_POST` field is `morph_triggers`, not the default `morph_animation`, to avoid confusion with the per-group setting.

---

## PHP Side

### Constants

| Constant | Value | Purpose |
|---|---|---|
| `SETTING_KEY` | `morph_animation` | Layer-settings key |
| `TRIGGERS_META` | `_wp3dconf_morph_triggers` | Post meta key |
| `MAX_TRIGGERS` / `MAX_TRACKS` | 50 / 200 | Bound what a saved or imported payload can make the frontend resolve |
| `EASINGS` | `linear`, `easeIn`, `easeOut`, `easeInOut` | Allowed easing names |

### Public static helpers

- `Morph_Animation::normalize( $raw, $allow_empty = false )` — reduce one animation to `{ duration, easing, tracks }` or `null`. Duration is clamped to 0.05–10; unknown easing becomes `easeInOut`.
- `Morph_Animation::normalize_triggers( $raw )` — reduce the whole triggers payload to `{ default, items }` or `null`.

### Payloads

**`window.wp3dconfEditorData.morphAnimation`** (editor):

```js
{ iconId: 'morph-animation', triggers: { default, items }, i18n: { ... } }
```

The strings live in the module rather than the editor's own i18n so Lite never carries them.

**`window.WP3dConf.morphAnimation`** (frontend):

```js
{
  groups:   { "<group uid>": "<animation id>" },   // only ids that exist
  triggers: { default, items }
}
```

`groups` is built from the **saved** `_wp3dconf_layer_settings` meta, not the layer payload, so a configurator that arrived through an import is cleaned the same way as one saved in the editor.

### Hooks used

| Hook | Use |
|---|---|
| `wp3dconf/editor/save_configurator` (20) | `normalize_saved_settings()` |
| `wp3dconf/frontend/canvas/after_canvas` | `render_trigger_dropdown()` prints the dropdown markup |

Markup contract: the bundle binds to `[data-wp3dconf-morph-trigger]`; option values (`data-value`) are the animation ids. Classes are `wp3dconf-morph-trigger*` — style them in your theme if needed.

---

## Frontend Runtime

`morph-animation.min.js` is an IIFE. It waits for `wp3dconf/frontend:ready` if the hook bus isn't there yet, then subscribes to:

| Host event | Used for |
|---|---|
| `wp3dconf/frontend/user/group_toggled` | Open/close rules |
| `wp3dconf/frontend/user/option_clicked` | Re-apply the group's animation |
| `wp3dconf/frontend/modelLoaded` | Bind the scene |
| `wp3dconf/frontend/replacementModelLoaded` | Re-bind after a model swap |

Internals (`common/_dev/modules/morph-animation/frontend/`):

- `rules.ts` — pure functions `decideOpen`, `decideClose`, `decideOption` returning `activate` / `deactivate` / `none`. Easy to unit-test; keys are `trigger:<id>`.
- `animator.ts` — `MorphAnimator`: per-key progress, `requestAnimationFrame` loop that only runs while something moves; writes morph influences only (the viewer already renders every frame).
- `layers.ts` — `nearestAnimatedGroup()`, with a loop guard for malformed layer trees.
- `dropdown.ts` — accessible listbox dropdown behaviour.

---

## Extending

- **Add another easing:** add the name to `Morph_Animation::EASINGS` and handle it in `easeProgress()` in `src/utils/morphEvaluate.ts`; add its label to the editor `i18n` and select.
- **Change limits:** `MAX_TRIGGERS` / `MAX_TRACKS` only bound the sanitizer; nothing else depends on them.
- **Rebuilding:** after editing anything under `common/_dev/modules/morph-animation/` or the shared `src/` files it uses, run the module build (`composer build-assets` / `common/_dev/scripts/build-modules.mjs`). The editor also needs rebuilding if the host stores (`use3DStore`, `useMorphTriggersStore`) change, since the module bundle receives them via `window.wp3dConf`.
- **Changing the saved shape:** there is no schema version on this data. Old configurators must keep loading, so add defensive defaults in `normalize()` / `normalize_triggers()` rather than changing field meanings.

---

## Known limitations (tracked as `@TODO`s)

- Easing and duration are per animation, not per movement; there is no separate "in" and "out" duration.
- Only morph targets are animated — GLB animation clips are not (the saved shape leaves room for it).
- Closing a group returns to rest; it does not fall back to a still-open parent group's animation.
- Animation edits are not in the editor's undo history, and there's no unsaved-changes warning on leaving the editor.
- Frustum culling is disabled on animated meshes instead of growing the bounding sphere.

---

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| Dialog says the model has no built-in animations | The loaded GLB has no morph targets. |
| Dialog says "Waiting for the model to load…" | Model still loading; reopen once it appears. |
| "The editor could not give this panel the loaded model" | Editor bundle out of date — rebuild editor assets and reload. |
| A movement shows a warning | Its part/target name isn't in the current model; it's skipped. Re-pick it. |
| Group plays nothing | The animation was deleted (assignment is cleared on save), or the animation has no movements (it's the resting pose). |
| Opening a group via Conditional Logic doesn't animate | By design — only visitor clicks drive group animations. |
| No dropdown on the frontend | No animations saved for that configurator. |
| Nothing moves for visitors with motion settings reduced | Reduced-motion applies the pose instantly instead of animating. |
