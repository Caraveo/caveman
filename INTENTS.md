# CAVEMAN scene-description API

The extension sends prompts to Claude, receives a small JSON scene description, validates each operation, and applies it through Blender's `bpy` API. Generated output never executes Python. Each operation is explicitly allowlisted and bounded.

## Supported operations

| Intent | Blender implementation |
| --- | --- |
| Create primitive | Cube, plane, UV sphere, icosphere, cylinder, cone, torus, and Suzanne |
| Create custom mesh | [`bpy.types.Mesh.from_pydata`](https://docs.blender.org/api/4.5/bpy.types.Mesh.html), with at most 1,200 vertices, 1,200 faces, and 16 vertices per face |
| Build an image shape layer | Create a closed relief mesh from a convex normalized image-space outline; front/back center fans and side walls make a push-pull solid, with X/Z following image left-right/up-down and Y controlling depth |
| Place and orient | Object location, rotation, and scale relative to the 3D cursor |
| Shape like a sculpt pass | Bounded edits to [`bpy.types.MeshVertex.co`](https://docs.blender.org/api/4.5/bpy.types.MeshVertex.html): inflate, draw, crease, and smooth; up to 64 strokes per mesh |
| Create paint texture | Generate a packed 64–512 px base-color image, simple UV islands, and up to 80 color strokes; connect it to the Principled BSDF and set it as Blender's paint canvas |
| Assign materials | Principled BSDF base color, metallic, and roughness |
| Mold/refine geometry | Bevel, subdivision, solidify, mirror, voxel remesh, and smooth modifiers; at most four modifiers per object |
| Join and cut solids | Create a [Boolean modifier](https://docs.blender.org/api/4.5/bpy.types.BooleanModifier.html) with the Exact solver and apply it through [`bpy.ops.object.modifier_apply`](https://docs.blender.org/api/4.5/bpy.ops.object.html); each ordered `UNION`, `DIFFERENCE`, or `INTERSECT` consumes its named operand, up to 60 operations |
| Build a skeleton | Create up to 32 named armature bones with parent links; bones are positioned relative to the 3D cursor |
| Bind meshes | Calculate normalized weights from each vertex's distance to the nearest bone segments and add armature modifiers |
| Read an image reference | Send a selected local JPEG, PNG, GIF, or WebP as visual data; translate its silhouette and adjoining polygon regions into mesh vertices/faces, then attach continuous forms through shared topology or Boolean unions |
| Generate a shape guide | Optional **Generate Image** checkbox asks Claude for polygon vertices, region roles, and normalized depth values, then rasterizes a transparent grayscale PNG locally with Blender's [`bpy.types.Image`](https://docs.blender.org/api/4.5/bpy.types.Image.html) pixel and save APIs |
| Build/reveal a model | Create each part in order, key its scale for a sequential reveal, and replay from the CAVEMAN panel |
| Capture a scene | Render a material-preview viewport screenshot of visible scene meshes and save it under the extension user's `captures` directory; the panel also has a manual capture button |
| Compare and refine | Send the scene screenshot, original prompt, and optional reference image to Claude; compare visible outlines and forms in corresponding views, then apply a revised scene plan with targeted mesh and sculpt changes for up to the configured number of edit phases |

Custom mesh vertex coordinates and sculpt stroke centers are local to each mesh. Image shape outlines use normalized image coordinates and must be convex; decompose a concave region into overlapping convex shape objects. Each shape produces a closed relief mesh with a triangular front fan, rear cap, and side walls; object X/Z map to image horizontal/vertical and Y is the depth axis. `shape_data.relief` and sculpt strokes push or pull the front surface. Regions that form one continuous solid should share topology inside one custom mesh or be joined with Boolean `UNION`; leave components separate only for visible gaps or joints. Texture stroke points use normalized UV coordinates from 0 to 1. For image references, the model first decomposes the view into an outer silhouette and adjoining shape regions, then compares the screenshot and reference visually and revises region outlines, depth, attachments, and local sculpt strokes to bring projected forms closer. This is vision-guided matching rather than a numerical pixel-error optimizer. Boolean operations are applied before existing target modifiers, so target bevel modifiers can round newly cut or joined edges. Each operand is removed after its Boolean operation; use a separate cutter mesh for every cut and don't include cutters in the rig. Rig endpoints use model coordinates relative to the 3D cursor. The optional top-level `rig` binds all remaining generated meshes by default; `objects` can narrow that list by object name.

## Prompt templates

- General model
- Furniture
- Product or prop
- Architecture
- Stylized character
- Image reference
- Speed Model

The **Enhanced Prompt** adds construction guidance. The **Speed Model** template places the main silhouette first, then major forms, then a few defining details. The generated timelapse follows object order, so each new part develops the design visibly.

**Screenshot Refinement** is enabled by default. When enabled, after the first build the feedback loop frames the generated meshes, captures a material-preview viewport screenshot with only that model visible, and sends the screenshot with the initial request and optional reference image to Claude. The **Visual Edit Phases** setting limits how many revised plans can be built; after the limit, a final screenshot review can still mark the current model complete. When disabled, CAVEMAN skips the post-build screenshot and review API call; an attached reference image still guides the initial build. Review screenshots are saved under the extension user data `captures` folder. The panel's **Capture Scene Screenshot** button saves a screenshot of visible scene meshes locally without sending it to Claude. Other Blender scene objects stay hidden only during capture and have their visibility restored immediately afterward.

**Generate Image** is off by default. Claude does not return raster images through its API; it returns a bounded JSON shape map with polygon outlines, region roles, and depth values. When enabled, CAVEMAN sends the model prompt, editable **Image Generation Prompt**, and optional local image to Claude for this shape map, then rasterizes a transparent black-and-white depth guide locally using Blender's Image API. The PNG is saved under extension user data `generated` and sent back to Claude as the primary shape reference for the full scene plan. A selected local reference image is also sent again as supplementary visual evidence. Only the existing Claude API key is required. The default additional prompt requests a centered orthographic grayscale depth-style image with raised and recessed forms, connected polygon regions, and no background, text, or shadows.

## Example response shape

```json
{
  "objects": [
    {
      "name": "Torso",
      "type": "uv_sphere",
      "location": [0, 0, 1.2],
      "rotation": [0, 0, 0],
      "scale": [0.7, 0.4, 1.0],
      "parameters": {"segments": 32, "rings": 16},
      "material": {"name": "Cloth", "color": [0.12, 0.3, 0.2], "roughness": 0.7},
      "sculpt": [{"mode": "INFLATE", "center": [0, 0, 0], "radius": 0.5, "strength": 0.1}],
      "texture": {"resolution": 256, "base_color": [0.12, 0.3, 0.2], "strokes": []},
      "modifiers": [{"type": "SUBSURF", "levels": 1}]
    },
    {
      "name": "Chest Plate Shape",
      "type": "shape",
      "location": [0, -0.2, 1.2],
      "scale": [0.8, 0.7, 0.5],
      "shape_data": {
        "outline": [[0.3, 0.2], [0.7, 0.2], [0.8, 0.5], [0.5, 0.9], [0.2, 0.5]],
        "relief": 0.12
      },
      "sculpt": [{"mode": "DRAW", "center": [0, -0.12, 0], "radius": 0.25, "strength": 0.08}]
    }
  ],
  "booleans": [
    {"target": "Torso", "operand": "Chest Plate Shape", "operation": "UNION"}
  ],
  "rig": {
    "bones": [
      {"name": "Root", "head": [0, 0, 0], "tail": [0, 0, 0.8]},
      {"name": "Spine", "head": [0, 0, 0.8], "tail": [0, 0, 1.8], "parent": "Root"}
    ]
  }
}
```

## Video-derived speed-model prompt

The reference [character-modeling timelapse](https://www.youtube.com/watch?v=n8Nmt8mOljo) alternates a full-body view with close work on anatomy, armor, and accessories, then adds material accents. The prompt adapts this pacing to editable meshes, sculpt passes, paint textures, and optional rigging supported by this extension.

> Build an editable, recognizable 3D model of **[subject]**. Establish the main body and silhouette first, add the largest secondary forms next, shape important contours with a few restrained sculpt strokes, then finish with distinctive accessories and simple painted color accents. Use custom meshes when primitives cannot capture the silhouette. Use Boolean unions to fuse parts that should read as one solid, and Boolean differences for holes, recesses, sockets, and bezels. Add bevel modifiers to soften manufactured cut edges. Keep proportions and part alignment coherent. Add a simple parented skeleton and bind the main meshes when the subject benefits from a rig. Keep the object, Boolean, stroke, texture, modifier, and bone counts within the declared limits. Order the parts so the model reads early and each later part adds a clear improvement. Return only the supported JSON scene description; never return executable Blender code.
