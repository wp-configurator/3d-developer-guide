# Preparing a Model for Custom Text (Blender)

**Audience:** 3D artists preparing a GLB for the Custom Text layer.
**Not for:** shop owners. This covers the model, not the plugin settings (see [custom-text.md](custom-text.md)).
**Applies to:** plugin v1.3.6
**Last updated:** 2026-10-30

Custom Text draws a shopper's words onto a **target mesh**. The plugin does not project text onto
your model. It draws the text onto a transparent image and swaps that image in as the texture of a
mesh you supply, so the mesh has to be built for it. If the mesh is wrong, the text lands in the
wrong place, stretched, mirrored, or fights with the base surface. No plugin setting fixes that
afterwards.

If you have prepared a model for Custom Image Upload, the setup is almost identical. See
[custom-image-upload-blender-guide.md](custom-image-upload-blender-guide.md). The differences are
called out below.

## How it works

The text is drawn on a **duplicate of the product mesh** that sits on top of the original. The
original keeps its own material. The duplicate carries only the text, with transparency everywhere
the text is not.

```
Base mesh (jersey)      -> normal material, never touched by the text
Text mesh (copy)        -> drawn text + alpha, sits a hair above the base
```

The plugin draws the text into a **1024 x 1024 square image** and uses it as the whole texture of
the text mesh. That square is the 0-1 UV space. Whatever part of the mesh's UVs falls inside the
square shows the text. Whatever falls outside shows nothing. The UV layout is the whole design.

### Where the text goes inside the square

The plugin needs to know how big the print area is. It finds out in this order:

1. **It measures the artwork already on the mesh.** If your text mesh has a texture with a default
   design (for example a placeholder name strip on a transparent background), the plugin finds the
   box that artwork occupies and fits the text inside it.
2. **If there is no usable texture**, it uses a centred square covering **85%** of the image.

Measuring beats the fallback, because you control the print area exactly. **Always give the text
mesh a transparent placeholder texture** with your print area painted on it.

Unlike Custom Image Upload, the text is fitted into the measured box as it is, **not into a square
around it**. A wide, short name strip stays wide and short, and the text fills it.

A measured area is ignored when it covers 95% or more of the image in both directions, as that is
treated as a full-bleed texture rather than a print area. Leave a margin around your placeholder.

## Workflow

### 1. Duplicate the mesh

Select the mesh the text will sit on (e.g. the jersey back) and duplicate it (`Shift+D`, then
`Esc`/right-click to cancel the move). Rename the copy so you can find it later, e.g.
`Jersey_Text_Back`.

### 2. One text mesh per text area

If the product has several independent text areas (back name, back number, sleeve), make **one
duplicate per area**. Each area has its own UV placement, and each is picked separately as a Target
Mesh in the editor.

- If the **same text** should appear in several places (front and back), one layer can target
  several meshes. They all show the same text.
- If each area needs **different text**, they need separate layers, so separate meshes.

### 3. Offset the text mesh to prevent z-fighting

A coplanar duplicate flickers against the base surface. Push the copy outward along its normals by
a very small amount:

- Edit Mode, select all, `Alt+S` (Shrink/Fatten), move slightly outward, or
- add a Displace modifier with a tiny strength and apply it.

Keep it small. Too large and the text visibly floats at silhouette edges and on curved areas.
Start around 0.001-0.002 units at real-world scale and check the model from grazing angles.

### 4. Build the material with alpha

Give the duplicate its own material:

- Base Color: a texture slot the plugin will replace with the drawn text.
- Alpha: driven by the texture's alpha channel.
- Blend mode: alpha blend/clip (Material Settings > Blend Mode), so transparent regions really are
  transparent in the exported GLB.

Confirm in the exported GLB that the material's `alphaMode` is `BLEND` or `MASK`. An opaque
material covers the whole product with a solid patch.

Put your **placeholder artwork** in that slot: a transparent PNG with the print area marked. The
plugin measures it, then replaces it as soon as a shopper types.

### 5. Adjust the UVs so the text lands correctly

This step decides where and how big the text appears.

1. Open the UV Editor on the text mesh.
2. Move and scale the UV islands so the **print area lines up with the 0-1 square**, and with the
   placeholder artwork you painted.
3. Everything outside the square is not printed.

**Match the square's proportions to the print area.** The image is square (1:1). If your print area
is physically wider than tall (a name strip), scale the UVs **non-uniformly** so the text is not
squashed. Verify with a checkerboard texture: squares must look like squares on the surface.

**Orientation matters.** Text is drawn the right way up in the image, and the plugin applies it
without flipping (the same as glTF's top-down UVs). The UVs must therefore have the top of the
print area at the top of the square. Check with a texture that has a letter on it, such as an
**F** or an arrow, to catch mirroring and rotation. A checkerboard alone cannot show this.

### 6. Keep clear of the edges

The plugin never draws text within **5% of the image edge** (about 51 pixels of 1024). This is
deliberate: the GPU repeats the outermost pixels when a mesh reaches past the texture, which would
smear any ink on the edge across the model.

- Do not put your print area right against the edge of the UV square. Text is pulled in to the 5%
  margin, so it is drawn a little smaller than your placeholder.
- Keep **stray UVs** (those outside the print area) pointing at empty, transparent pixels. Parts of
  the mesh with UVs outside the square can pick up the repeated edge pixels.

### 7. Keep the full mesh

**Do not delete the geometry you do not need on the duplicate**, even if the text area is a small
patch on a big product. Hide the unused parts with the alpha map instead.

Deleting faces from the duplicate can cause a colour mismatch against the original mesh. Keeping the
full mesh keeps shading, normals and lighting consistent with the base.

## Curved surfaces

Text can be bent, rotated and moved by the shopper, but it is only ever drawn **flat in the UV
square**. The model's surface does the rest of the work.

- On a curved surface (a mug, a bottle, a sleeve), the text wraps with the surface. Make the UVs
  there an even, **unwrapped** layout with no stretching, or the text will stretch with them.
- Text that wraps all the way around a cylinder needs a UV layout that is a continuous strip with
  no seam in the middle of the print area.

## Naming

The editor lists meshes by name when the shop owner picks a **Target Mesh**. Use clear names such as
`Jersey_Name_Back` or `Mug_Text_Front`. Avoid `Cube.001`.

## Check before export

Test with a checkerboard, then a texture with a letter **F**, in the text mesh's colour slot:

- [ ] Squares appear square on the surface (no stretching)
- [ ] The **F** reads correctly: not mirrored, not rotated, not upside down
- [ ] The print area lands where you intended
- [ ] No flicker or shimmer against the base (z-fighting)
- [ ] Nothing outside the print area is visible
- [ ] Material exports with alpha (`BLEND`/`MASK`)
- [ ] A transparent placeholder texture with the print area marked is on the mesh
- [ ] The placeholder leaves a margin (covers less than 95% of the image) and stays off the 5% edge
- [ ] Full duplicate mesh kept, no deleted faces
- [ ] Mesh names are clear enough to find in the Target Mesh dropdown

## Common problems

| Symptom | Cause |
|---|---|
| Text is stretched or squashed | UV area does not match the physical print area's proportions |
| Text is mirrored or upside down | UV island flipped or rotated relative to the 0-1 square |
| Text is tiny in the middle of the model | No placeholder texture found, so the 85% centred fallback was used; or the placeholder is much larger than the real print area |
| Text ignores the print area | Placeholder covers 95% or more of the image, so it was treated as full-bleed |
| Flicker or patchy overlap with the base | Text mesh is not offset from the base |
| Solid patch instead of transparency | Material exported opaque |
| Text colour looks different from the product | Faces were deleted from the duplicate, or the material differs from the base |
| A smear or line of colour runs off the text area | Stray UVs outside the square pick up repeated edge pixels, or the print area touches the edge |
| Text looks blurry up close | Plugin draws a 1024 pixel square. A very small UV area on a large surface magnifies it, so give the text mesh a bigger share of the UV square |
| Message "not set up correctly" on the site | No Target Mesh was picked in the editor |

## Handing over to the shop owner

Give the shop owner:

- the GLB,
- the **names of the text meshes**, and what each one is for,
- the intended text area for each (e.g. *a name strip, about 300 x 60 mm; keep it under 12 letters*), so they can set **Max Length** and **Input Type** sensibly.
