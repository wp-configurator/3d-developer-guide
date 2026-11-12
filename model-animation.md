# Model Animation - User Guide

**Audience:** shop owners setting up animations in the configurator editor.
**Not for:** 3D artists (see [model-animation-blender-guide.md](model-animation-blender-guide.md)) or developers (see [model-animation-developer-guide.md](model-animation-developer-guide.md)).
**Applies to:** plugin v1.3.5
**Last updated:** 2026-11-12

Model Animation moves parts of your 3D model: open a bag to show its inside, lift a lid, pull out a handle. Your customers see it in two ways:

- **When they open a group.** A group can play an animation when a customer opens it, and play it back when they close it or open another group.
- **From a dropdown over the model.** Customers switch the model between named poses, such as *Closed* and *Open*.

The plugin only moves what the model already contains, called shape keys. Your 3D artist adds them; see [model-animation-blender-guide.md](model-animation-blender-guide.md). If the model has none, there is nothing to animate and no setting can add them.

> **Pro only.** Model Animation is not in the Lite version.

📷 GIF: product closed, then open.

---

## How it works

Each animation is a named pose, such as *Closed* or *Open*. An animation is made of one or more **movements**. A movement changes one **part** of the model (for example the lid) by a set amount, at a set time.

```text
Animation "Open"
  Movement 1: Part Lid,  Movement Open  -> 0 to 1, during the first 60%
  Movement 2: Part Flap, Movement Lift  -> 0 to 1, during the last 50%
```

| Term | Meaning |
|---|---|
| Animation | A named pose or motion, such as *Closed* or *Open* |
| Part | A piece of your model, such as *Lid* |
| Movement | What that part does, such as *Open* |
| Resting pose | The model with nothing moved |

---

## Before you start

- **Model Animation** is turned on under **wp3dconf → Modules**.
- Your model has movements built in. If it doesn't, the Animations window says: "This 3D model has no built-in animations, so there is nothing to animate."

📷 Modules page with Model Animation enabled.

---

## Setting it up

1. Open a configurator whose model has movements, and click the **Animations** (play) icon in the editor toolbar. A window opens that you can drag around.

   📷 Toolbar icon and empty panel.

2. Click **Add Animation**. Each animation has its own tab with:
   - **Name** — shown to customers in the dropdown.
   - **Play when the page loads** — makes this the animation customers see first (marked *On load*). Only one animation can have this.
   - **Movements** — what moves; see below.
   - **Duration** (0.05 to 10 seconds) and **Easing** (Linear, Ease in, Ease out, Ease in and out).

   📷 Animation tab with these fields.

3. Click **Save**, or save the configurator as usual. The animations are saved with it.
4. To play an animation when a group opens: select the **group** or **sub group**, open its **Animation** settings, and choose the animation under **Trigger**.

   📷 Group settings with Animation → Trigger.

### Movements

Each movement moves one part of the model:

| Field | What it does |
|---|---|
| **Part** | The part of the model to move. |
| **Movement** | Which of that part's movements to play. |
| **From / To** | How far the movement goes, from 0 (not at all) to 1 (fully). |
| **Start / End** | When it happens within the animation, from 0 (the start) to 1 (the end). Use this to play movements one after another, for example the lid lifts from 0 to 0.5, then the flap opens from 0.5 to 1. |

📷 Movements list with Amount and Timing columns.

Each part and movement can only be used once per animation; a duplicate is removed when you save.

Movements are matched to the model by name, so they keep working if your artist re-exports the model, as long as the names stay the same. If a part or movement is renamed or removed, the editor shows a warning on that row and customers won't see it move. Pick it again to fix it.

### A "Closed" animation

An animation with **no movements** is the resting pose: it puts the model back to how it started. Create one called *Closed* and tick **Play when the page loads**, so customers start from the closed model and can always get back to it from the dropdown.

---

## What customers see

Only **one animation shows at a time**: the last one started. Starting another plays the first one back. If a customer clicks while something is moving, it carries on smoothly from where it is.

### Groups

- Opening a group with an animation plays it.
- Opening a group with no animation puts the model back to rest, unless that group is inside a group whose animation is showing.
- Closing the group whose animation is showing plays it back.
- Clicking an option inside an animated group shows that group's animation again, if the dropdown or another group had changed it.

Animations only react to a customer's own clicks. A group opened automatically, for example by Conditional Logic or when the page loads, doesn't play its animation.

### Dropdown

The dropdown appears over the model as soon as the configurator has at least one animation. It can be used with the keyboard as well as the mouse.

📷 Dropdown on the frontend, closed and open.

---

## Limits and good to know

- Up to 50 animations per configurator, and 200 movements per animation.
- Customers who have asked their device to reduce motion see the model change pose straight away, without the movement.
- If another option swaps in a different model part, it animates too, as long as its part and movement names match, and it starts in the pose already showing.
- Only the movements built into the model are animated. Bones and animation clips are not.
- Animation changes are not in the editor's undo history.

---

## Check before publishing

- [ ] Model Animation is turned on and the model has movements
- [ ] Every animation has a clear name
- [ ] A resting pose (*Closed*) exists and is set to play when the page loads
- [ ] Only one animation is marked **Play when the page loads**
- [ ] No movement shows a warning
- [ ] Each animation looks right in the preview
- [ ] Groups have the correct **Trigger** selected
- [ ] Saved, and tested on the frontend by clicking groups and using the dropdown

---

## Troubleshooting

| Problem | What to check |
|---|---|
| "This 3D model has no built-in animations, so there is nothing to animate." | The model has no movements. Ask your 3D artist to add them, and share the Blender guide. |
| "Waiting for the model to load..." | The model is still loading. Close the window and open it again once the model appears. |
| "The editor could not give this panel the loaded model." | Reload the editor. If it keeps happening, contact your developer. |
| A movement row shows a warning | That part or movement is no longer in the model, so it is skipped. Pick it again. |
| Animations stopped working after uploading a new model | The part or movement names changed. Pick them again. |
| Two movements fight each other | The same part and movement is used twice. Only one is kept when you save. |
| A group plays nothing | The animation was deleted, or it has no movements (it's the resting pose). |
| A group opened by Conditional Logic doesn't animate | This is by design: animations only follow a customer's own clicks. |
| There is no dropdown on the page | The configurator has no saved animations. |
| Nothing moves for some customers | Their device is set to reduce motion, so the pose changes without moving. |

---

## Reference screenshots

- Left, the Animations window: an animation with its movements, Amount and Timing.
- Right, the frontend: the dropdown over the 3D view in two poses.
