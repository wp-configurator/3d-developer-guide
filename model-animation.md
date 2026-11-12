# Model Animation - User Guide

**Audience:** shop owners setting up animations in the configurator editor.
**Not for:** 3D artists (see [model-animation-blender-guide.md](model-animation-blender-guide.md)) or developers (see [model-animation-developer-guide.md](model-animation-developer-guide.md)).
**Applies to:** plugin v1.3.5
**Last updated:** 2026-11-12

Model Animation moves parts of your 3D model: open a bag to show its inside, lift a lid, pull out a handle. Your customers see it in two ways:

- **When they open a group.** A group can play an animation when a customer opens it, and play it back when they close it or open another group.
- **From a dropdown over the model.** Customers switch the model between named poses, such as *Closed* and *Open*.

The model must already contain the movements, called shape keys. Your 3D artist adds them; see [model-animation-blender-guide.md](model-animation-blender-guide.md).

> **Pro only.** Model Animation is not in the Lite version.

---

## Setting it up

1. Turn on **Model Animation** under **wp3dconf → Modules**.
2. Open a configurator whose model has shape keys, and click the **Animations** (play) icon in the editor toolbar. A window opens that you can drag around.
3. Click **Add Animation**. Each animation has its own tab with:
   - **Name** — shown to customers in the dropdown.
   - **Play when the page loads** — makes this the animation customers see first (marked *On load*). Only one animation can have this.
   - **Movements** — what moves; see below.
   - **Duration** (0.05 to 10 seconds) and **Easing** (Linear, Ease in, Ease out, Ease in and out).
4. Click **Save**, or save the configurator as usual. The animations are saved with it.
5. To play an animation when a group opens: select the **group** or **sub group**, open its **Animation** settings, and choose the animation under **Trigger**.

### Movements

Each movement moves one part of the model:

| Field | What it does |
|---|---|
| **Part** | The part of the model to move. |
| **Movement** | Which of that part's movements to play. |
| **From / To** | How far the movement goes, from 0 (not at all) to 1 (fully). |
| **Start / End** | When it happens within the animation, from 0 (the start) to 1 (the end). Use this to play movements one after another, for example the lid lifts from 0 to 0.5, then the flap opens from 0.5 to 1. |

Each part and movement can only be used once per animation; a duplicate is removed when you save.

Movements are matched to the model by name, so they keep working if your artist re-exports the model, as long as the names stay the same. If a part or movement is renamed or removed, the editor shows a warning on that row and customers won't see it move. Pick it again to fix it.

### A "Closed" animation

An animation with **no movements** puts the model back to how it started. Create one called *Closed* and tick **Play when the page loads**, so customers start from the closed model and can always get back to it from the dropdown.

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

### Good to know

- Customers who have asked their device to reduce motion see the model change pose straight away, without the movement.
- If another option swaps in a different model part, it is put in the pose already showing.

---

## Troubleshooting

| Problem | What to check |
|---|---|
| The Animations window says the model has no built-in animations | The model has no shape keys. Ask your 3D artist to add them. |
| The window says "Waiting for the model to load…" | The model is still loading. Close the window and open it again once the model appears. |
| The window says "The editor could not give this panel the loaded model" | Reload the editor. If it keeps happening, contact your developer. |
| A movement row shows a warning | That part or movement is no longer in the model. Pick it again. |
| A group plays nothing | The animation was deleted, or it has no movements (it's the "Closed" pose). |
| A group opened by Conditional Logic doesn't animate | This is by design: animations only follow a customer's own clicks. |
| There is no dropdown on the page | The configurator has no saved animations. |
| Nothing moves for some customers | Their device is set to reduce motion, so the pose changes without moving. |
