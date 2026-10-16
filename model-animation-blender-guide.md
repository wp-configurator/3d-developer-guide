# Preparing a Model for Model Animation (Blender)

**Audience:** 3D artists preparing a GLB whose parts move with the Model Animation module.
**Not for:** shop owners. This covers the model, not the plugin settings (see [model-animation.md](model-animation.md)).
**Applies to:** plugin v1.3.5
**Last updated:** 2026-10-16

Model Animation moves parts of the model by blending **shape keys** (called *morph targets* in
glTF and three.js). The plugin does not animate bones or play clips. It changes how strongly each
shape key is applied, from the editor's settings, so the motion you want has to be sculpted into
the shape keys beforehand. If a key is missing, misnamed or exported wrongly, the shop owner has
nothing to pick and the part stays still. No plugin setting fixes that afterwards.

## How it works

Each shape key stores a *deformed copy* of a mesh, relative to its rest shape. The plugin
animates a key's **influence** from 0 (rest shape) to 1 (fully deformed) and back.

```
Bag mesh  (base shape)     -> closed bag
Shape key "Open" (0 -> 1)  -> bag with the flap lifted and the interior exposed
```

The shop owner picks a **Part** (the mesh name) and a **Movement** (the shape key name) in the
editor, and sets how far (`From`/`To`, 0-1) and when (`Start`/`End`) it moves. Several keys can be
chained or run together in one animation. Everything is matched by **name**, never by index.

## Workflow

### 1. Model the rest pose as the base shape

The base shape (`Basis`) is what visitors see before anything plays and what the model returns to
when an animation is reversed. Make it the product's default state, e.g. the **closed** bag.

Leave every shape key's value at **0** when you export. The plugin treats the exported value as the
model's starting pose and returns to it when an animation rewinds.

### 2. Add one shape key per movement

On the mesh that moves (Object Data properties > Shape Keys), add `Basis` first, then one key per
movement. Sculpt or edit vertices in each key to the fully-moved pose.

- Split independent movements into **separate keys** (lid, flap, strap). The editor can then time
  them one after another or together. Combining them in one key forces them to move as one.
- Keep the **Relative to** setting on `Basis` (the default). glTF supports only relative keys.
- Leave each key's slider range at **0 to 1**. The plugin clamps influence to 0-1, so values
  beyond that range, or negative values, are never reached.

### 3. Name meshes and shape keys clearly

Shop owners choose from these names in the **Part** and **Movement** dropdowns, and saved
configurators find them again **by name**.

- Use short, readable names: `Lid` and `Open`, `Flap` and `Lift`.
- Give **every moving mesh a unique name.** If two meshes share a name, only the first is listed.
- **Do not leave moving meshes unnamed.** An unnamed mesh cannot be referenced and is skipped.
- **Do not rename** meshes or keys after the shop owner has built animations. A renamed part or
  movement is flagged in the editor and skipped on the frontend until it is picked again.

A mesh with several materials is exported as several primitives, and **each primitive is listed as
its own part, each with its own shape keys.** Keep the shape key names identical across the
primitives of one mesh so every piece moves together, and check the generated names in the editor.

### 4. Arrange the geometry so the motion reads well

Shape keys interpolate each vertex in a **straight line** between rest and moved positions. This is
not a rotation around a hinge.

- A small rotation (up to roughly 45 degrees) looks fine.
- A large swing, such as a lid opening 120 degrees, **shrinks and squashes** partway through
  because vertices cut across the arc instead of following it.

For big swings, split the movement across two or three keys (`Open_1` at 45 degrees, `Open_2` at 90
degrees, `Open_3` at 120 degrees). The shop owner then chains them with `Start`/`End` timing. You can
also add a corrective key that bulges the middle of the arc.

If something is revealed, such as an interior, **model it inside the base shape** (hidden by the
closed state) rather than adding it from nowhere. Every key must have the **same vertices** as the
base, so you cannot add or delete geometry in a key.

### 5. Do not rely on bones or animation clips

Armature or action animations in the file are **not** played by this module. Convert movements you
want into shape keys. If your rig is the only way to pose the part, bake the final pose into a
shape key and use that.

### 6. Check culling and framing

The viewer fits the camera using the model's **posed** vertices (the rest shape). Keep the model at
rest when you export so the framing matches what visitors see first. Parts that move outside the
rest bounds are not clipped at the screen edge.

### 7. Export the GLB with shape keys on

In **File > Export > glTF 2.0** (`.glb`):

- **Data > Mesh > Shape Keys:** enabled. If this is off, the moving parts export with no keys at all.
- **Data > Mesh > Shape Key Normals:** enabled if the surface shading should follow the deformation
  (recommended for smooth surfaces); otherwise normals stay on the rest shape.
- **Data > Mesh > Apply Modifiers:** avoid on meshes with shape keys. Applying modifiers on a
  mesh that has shape keys can drop the keys on export. Apply the modifiers yourself first, or turn
  the modifiers off on those meshes.
- **Animation:** nothing to enable. Shape key animation clips are ignored by the plugin.

Each shape key stores a full copy of the affected vertices, so many keys on a dense mesh grow the
file quickly. Keep keys to the meshes that move and avoid unnecessary resolution on them.

## Swapped models

If the shop owner uses a layer that swaps a mesh for another model, the swapped-in model is
animated too, **as long as its meshes and shape keys use the same names** as the original.
A replacement with different names animates nothing.

## Check before export

- [ ] `Basis` is the default state of the product (e.g. closed)
- [ ] Every shape key value is **0** at export
- [ ] One shape key per independent movement; slider range 0-1
- [ ] Moving meshes have **unique, readable names**; none unnamed
- [ ] Shape keys on all primitives of one mesh use the same names
- [ ] Large rotations are split into stages, or corrected, so nothing collapses mid-motion
- [ ] Nothing depends on bones or animation clips
- [ ] Shape Keys is enabled in the export, and Apply Modifiers is not used on morphed meshes
- [ ] Re-importing or opening the GLB shows the keys (e.g. in the Blender importer or a glTF viewer)

## Common problems

| Symptom | Cause |
|---|---|
| Editor says the model has no built-in animations | The GLB has no shape keys: Shape Keys was off at export, or modifiers dropped them |
| A mesh is missing from the **Part** list | The mesh is unnamed, or shares a name with another mesh that is listed first |
| Part is listed but **Movement** is empty | That mesh has no named shape key other than `Basis` |
| Part twitches or only half the model moves | A multi-material mesh exported as several primitives with different key names |
| Motion squashes or shrinks mid-swing | A large rotation in one shape key, as vertices move in straight lines; split it into stages |
| Model appears far smaller or off-centre in the editor | The model was exported with shape keys set to a non-zero value |
| An existing animation stopped working after a re-export | A mesh or shape key was renamed, so the saved name no longer matches |
| A swapped-in model does not move | Its meshes or shape keys are named differently from the original |
