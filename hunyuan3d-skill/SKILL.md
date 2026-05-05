---
name: hunyuan3d-blender
description: "Use this skill when generating a 3D model from an image or text prompt via Hunyuan3D MCP tools and importing the result into Blender. Trigger whenever mcp__hunyuan3d tools are involved OR when the user says 'generate from image', 'generate from text', 'AI 3D model', or 'Hunyuan3D'."
---

# Hunyuan3D → Blender Skill

## Pipeline Order

1. `server_health` — stop if unreachable
2. Build Hunyuan3D prompt — geometry-focused only, no color, no style words
3. `generate_3d_async` — kick off generation
4. Build scene while waiting — base disc + lights + camera only
5. `check_generation_status` — poll until complete
6. Convert path if on Windows
7. Import the GLB
8. Measure bounds — derive all numbers from real values
9. Scale, centre, seat on base disc
10. Shade smooth
11. Apply color and material in Blender with mandatory verification
12. Final verification against user request
13. Screenshot to confirm

---

## Critical Rules

- **One object per scene.** Hunyuan3D generates exactly one mesh. The scene contains exactly one HY3D hero object.
- **Hunyuan3D is the only 3D modeler.** Never create any 3D geometry in Blender — not primitives, not curves, nothing. Blender's role is strictly: import, place, color, light, camera.
- **Never guess scale or position.** Every number must come from measuring real bounding box values after import. No hardcoded offsets.
- **Never auto-clear the scene.** If the scene needs clearing, ask the user first.
- **Never use raw Linux paths in Blender on Windows.** Convert first — see Path Conversion below.

---

## Hunyuan3D Prompt Rules — Shape Only, No Color

**Describe geometry and form only. Never describe color, material, or texture in the prompt.**

- Focus on shape: structure, proportions, surface detail, physical form.
- Avoid open-ended style or setting words ("fantasy", "magical", "cute", "tropical") — these cause Hunyuan3D to hallucinate extra geometry or unwanted decorations.
- Be specific and restrictive about what should NOT appear when hallucination risk is high.
- All color, texture, and material work happens in Blender after import — never in the Hunyuan3D prompt.

**Prompt structure to follow:**
- What the object IS (noun + key physical traits)
- Key structural details (shape of parts, proportions)
- Explicit exclusions if needed ("no vegetation", "no extra parts", "standalone object only")

**Example — Good:** `"a detailed wooden chair, four straight legs, flat seat, vertical back slats, standalone object, no floor, no background"`
**Example — Bad:** `"a cozy rustic chair in a warm cottage setting with cushions"`

The good prompt describes geometry. The bad prompt describes a scene and style — both of which cause hallucination.

---

## Strict Object Restriction — CRITICAL

**After the HY3D mesh is imported, NO additional Blender geometry may be created under any circumstance.**

- The HY3D mesh is the deliverable. Its form must not be altered or overshadowed by any Blender geometry.
- Do not add decorations, props, layers, or supporting shapes based on assumptions or inferred intent from the user's description.
- The user's description is input for the Hunyuan3D prompt — it is not a Blender modeling instruction.

**The ONLY objects permitted in the final scene without explicit user instruction:**
- The HY3D mesh itself
- The base display disc
- Three lights
- One camera

Nothing else. Ever.

---

## While Waiting — Scene Setup

While Hunyuan3D is generating, build exactly three things:

**Base Disc:**
- Bevelled cylinder, radius `1.4`, depth `0.05`
- Neutral matte material — `#CCCCCC`, roughness `0.9`
- Shade smooth via `poly.use_smooth = True` on polygons — never `bpy.ops.object.shade_smooth`
- Name it `"Base"`
- Place at `location=(0, 0, -0.025)` so its top surface is exactly at `z=0`

**Lighting — 3-Point Rig:**
- Key light: area, warm `(1.0, 0.95, 0.8)`, energy `600`, front-right `(4, -3, 6)`
- Fill light: area, cool `(0.7, 0.85, 1.0)`, energy `250`, front-left `(-4, -2, 4)`
- Rim light: area, accent `(1.0, 0.9, 0.6)`, energy `350`, behind `(0, 5, 5)`

**Camera:**
- 85mm lens
- 3/4 hero angle, positioned to frame the full model on the base
- Set as `scene.camera`

---

## Path Conversion — Windows + WSL

HY3D runs in WSL and returns a Linux path. Blender on Windows cannot read it directly. Always convert before importing:

```python
import sys

def linux_to_windows_wsl(p):
    return "\\\\wsl.localhost\\Ubuntu" + p.replace("/", "\\")

glb_path = linux_to_windows_wsl(linux_path) if sys.platform == "win32" else linux_path
```

If the WSL distro is not `Ubuntu`, replace with the correct name (e.g. `Ubuntu-22.04`).

---

## Import Rules

- Deselect all objects before importing using a loop — `obj.select_set(False)` — never `bpy.ops.object.select_all`.
- Import the GLB using `bpy.ops.import_scene.gltf(filepath=glb_path)`.
- Collect only `MESH` type objects from `bpy.context.selected_objects` after import.
- If the result is empty — stop immediately and raise a clear error. Do not proceed silently.
- Rename the imported mesh to `"HY3D_Model"`.

---

## Placement Rules — Real Numbers Only

**Every placement decision must be derived from measured bounding box values. No guessing. No hardcoded offsets.**

```python
# Measure bounds from world-space vertices
verts_world = [obj.matrix_world @ v.co for v in obj.data.vertices]
xs = [v.x for v in verts_world]
ys = [v.y for v in verts_world]
zs = [v.z for v in verts_world]
min_x, max_x = min(xs), max(xs)
min_y, max_y = min(ys), max(ys)
min_z, max_z = min(zs), max(zs)
width  = max_x - min_x
depth  = max_y - min_y
height = max_z - min_z
```

**Scale:**
- Target height = `2.0` units
- `scale_factor = 2.0 / height`
- Apply uniformly: `obj.scale = (scale_factor, scale_factor, scale_factor)`
- Apply transform: `bpy.ops.object.transform_apply(scale=True)`
- Re-measure bounds after scaling

**Centre on X/Y:**
- `obj.location.x -= (min_x + max_x) / 2`
- `obj.location.y -= (min_y + max_y) / 2`

**Seat on base disc:**
- Base disc top surface is at `z=0`
- `obj.location.z -= min_z` — this places the model bottom exactly at `z=0`
- Apply transform: `bpy.ops.object.transform_apply(location=True)`

**Re-measure and print after all transforms:**
```python
print(f"Final height: {height:.4f}")
print(f"Bottom z:     {min_z:.4f}  (should be 0.0)")
print(f"Centre x:     {(min_x+max_x)/2:.4f}  (should be ~0.0)")
print(f"Centre y:     {(min_y+max_y)/2:.4f}  (should be ~0.0)")
print(f"Scale factor: {scale_factor:.4f}")
```

**Check orientation:**
- If the model faces away from camera, rotate on Z axis to face front.

**Shade smooth:**
```python
for poly in obj.data.polygons:
    poly.use_smooth = True
obj.data.update()
```

---

## Material & Color Rules

- All coloring and texturing happens in Blender after import — never in the Hunyuan3D prompt.
- Clear all existing GLB baked materials before assigning anything new: `obj.data.materials.clear()`
- Create a fresh material with a new Principled BSDF node tree.
- Match base color and roughness exactly to the user's specified values.
- If the user did not specify a color, use a sensible neutral default and note what was chosen.

### Mandatory Material Verification

After assigning every material, run all three checks:

```python
# 1. Confirm material is assigned
assert obj.data.materials[0].name == mat.name, "Material not assigned!"

# 2. Confirm node connectivity
nodes = mat.node_tree.nodes
links = mat.node_tree.links
bsdf   = nodes.get("Principled BSDF")
output = nodes.get("Material Output")
assert bsdf   is not None, "Principled BSDF node missing!"
assert output is not None, "Material Output node missing!"
connected = any(l.from_node == bsdf and l.to_node == output for l in links)
assert connected, "Principled BSDF not connected to Material Output!"

# 3. Print color confirmation
color = bsdf.inputs["Base Color"].default_value
print(f"Material '{mat.name}' verified — RGB: ({color[0]:.3f}, {color[1]:.3f}, {color[2]:.3f})")
```

If any check fails — delete the material, create it from scratch, and re-verify. Never patch existing nodes.

---

## Final Verification — MANDATORY Before Screenshot

Check every item below. Print PASS or FAIL for each. Fix all FAILs before proceeding.

```python
print("=== FINAL VERIFICATION ===")
print(f"Single HY3D object only:       PASS/FAIL")
print(f"No extra Blender geometry:     PASS/FAIL")
print(f"Model seated on base (z=0):    PASS/FAIL")
print(f"Model centred on X/Y:          PASS/FAIL")
print(f"Material verified:             PASS/FAIL")
print(f"Color matches user request:    PASS/FAIL")
print(f"Scene objects = HY3D + Base + 3 Lights + Camera only: PASS/FAIL")
print("==========================")
```

Do not take the screenshot until all checks are PASS.

---

## Expected Final Scene

```
Scene
├── Base          ← bevelled disc, z top = 0, neutral material
├── Key_Light     ← area, warm
├── Fill_Light    ← area, cool
├── Rim_Light     ← area, accent
├── Camera        ← scene.camera, 85mm
└── HY3D_Model    ← single mesh, bottom at z=0, centred, material verified
```

This is the complete scene. Nothing else belongs here.

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| `RuntimeError: Please select a file` | Missing path conversion — run `linux_to_windows_wsl()` |
| `imported = []` | Wrong WSL distro name, or file not yet written |
| `poll() failed, context is incorrect` | Use `obj.select_set(False)` loop instead of `bpy.ops.object.select_all`; use `poly.use_smooth` instead of `bpy.ops.object.shade_smooth` |
| Model too large or tiny | Re-derive scale factor from actual measured Z bounds |
| Model faces wrong way | Rotate on Z axis after seating |
| Model floating above base | Re-check `min_z` measurement and `obj.location.z -= min_z` step |
| Model sunk into base | Same — re-measure after every transform apply |
| Floating mesh fragments | Edit Mode → Merge by Distance |
| Material renders white in Cycles | Clear all materials first — `obj.data.materials.clear()` — then assign fresh Principled BSDF |
| Material verification assertion fails | Delete material entirely, recreate from scratch, re-verify |
| Hunyuan3D hallucinated extra geometry | Prompt was too open-ended — add explicit exclusions and re-generate |
