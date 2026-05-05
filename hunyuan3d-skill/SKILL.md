---
name: hunyuan3d-blender
description: "Use this skill when generating a 3D model from an image or text prompt via Hunyuan3D MCP tools and importing the result into Blender. Trigger whenever mcp__hunyuan3d tools are involved OR when the user says 'generate from image', 'generate from text', 'AI 3D model', or 'Hunyuan3D'."
---

# Hunyuan3D → Blender Skill

Full pipeline: health check → generate mesh via Hunyuan3D → prepare scene (base + lights + camera only) → import → seat → material → done.

---

## CRITICAL RULES — Read Before Writing Any Code

### ❌ Never Do This
- **Never build a procedural version** of the requested model while waiting for generation. The HY3D mesh IS the hero object. Do not create a duplicate character, creature, or prop from Blender primitives.
- **Never place two hero characters** in the scene unless the user explicitly requests multiple models.
- **Never use raw Linux paths** with `import_scene.gltf` when Blender runs on Windows. Convert first (see Path Conversion below).
- **Never guess scale or position.** Always measure bounds after import and derive placement from real numbers.

### ✅ Always Do This
- While generation runs: build **only** base platform + lighting rig + camera.
- Import the HY3D mesh, seat it on the base, apply material — that's the complete scene.
- After every scale or position change, verify bounds again before finalising.

---

## Phase 1 — Health Check

Always run first. Do not proceed if unreachable.

```python
server_health()   # must return OK before continuing
```

---

## Phase 2 — Choose Quality & Generate

| Use case | octree_resolution | num_inference_steps | approx time |
|---|---|---|---|
| Quick draft / iteration | 128 | 5 | ~15s |
| Balanced | 256 | 30 | ~60s |
| Final quality | 384 | 50 | ~2 min |

**From text prompt (async — always use for 256+ resolution):**
```python
uid = generate_3d_async(
    text="a cute stylized fantasy dragon, large head, chubby body, folded wings, idle pose",
    seed=42,
    octree_resolution=256,
    num_inference_steps=30,
    guidance_scale=6.0,
)
# Poll every 30s
status = check_generation_status(uid=uid)
# Returns Linux path when done: /home/user/Hunyuan3D-2/gradio_cache/<uid>.glb
```

**From image (preferred when reference exists):**
```python
uid = generate_3d_async(
    image_path="/abs/path/to/reference.png",
    seed=42,
    octree_resolution=256,
    num_inference_steps=30,
    guidance_scale=6.0,
)
```

Output is always a `.glb` in `gradio_cache/`. The returned path is a **Linux path** — convert before use in Blender (see below).

---

## Phase 3 — Prepare Scene WHILE Waiting

While generation runs, build **only** these three things — nothing else:

### 3a. Base Platform
```python
import bpy
# Simple bevelled disc — the HY3D mesh will sit on top
bpy.ops.mesh.primitive_cylinder_add(vertices=64, radius=1.4, depth=0.15, location=(0, 0, -0.075))
base = bpy.context.active_object
base.name = "Base"
mod = base.modifiers.new("Bevel", "BEVEL")
mod.width = 0.05; mod.segments = 3
bpy.ops.object.modifier_apply(modifier="Bevel")
bpy.ops.object.shade_smooth()
# Apply a material to taste
```

### 3b. Lighting Rig (3-point)
```python
import math
# Key light
bpy.ops.object.light_add(type='AREA', location=(3.5, -2.5, 4.5))
key = bpy.context.active_object; key.name = "Key_Light"
key.data.energy = 800; key.data.size = 3.0
key.rotation_euler = (math.radians(52), 0, math.radians(38))

# Fill light
bpy.ops.object.light_add(type='AREA', location=(-3.0, 2.0, 2.5))
fill = bpy.context.active_object; fill.name = "Fill_Light"
fill.data.energy = 250; fill.data.size = 4.0
fill.rotation_euler = (math.radians(48), 0, math.radians(-42))

# Rim light
bpy.ops.object.light_add(type='AREA', location=(-0.5, 4.0, 3.5))
rim = bpy.context.active_object; rim.name = "Rim_Light"
rim.data.energy = 300; rim.data.size = 2.0
rim.rotation_euler = (math.radians(-35), 0, math.radians(10))
```

### 3c. Camera
```python
bpy.ops.object.camera_add(location=(4.5, -4.0, 3.2))
cam = bpy.context.active_object; cam.name = "Camera"
cam.rotation_euler = (math.radians(65), 0, math.radians(49))
cam.data.lens = 85
bpy.context.scene.camera = cam
```

---

## Phase 4 — Path Conversion (Windows + WSL)

Blender on **Windows** cannot read raw Linux paths. Always convert before importing:

```python
def linux_to_windows_wsl(linux_path: str) -> str:
    """Convert /home/user/... to \\wsl.localhost\Ubuntu\home\user\..."""
    return "\\\\wsl.localhost\\Ubuntu" + linux_path.replace("/", "\\")

# Example
linux_path = "/home/laxmi/Hunyuan3D-2/gradio_cache/abc123.glb"
win_path   = linux_to_windows_wsl(linux_path)
# win_path = \\wsl.localhost\Ubuntu\home\laxmi\Hunyuan3D-2\gradio_cache\abc123.glb
```

If Blender runs natively on **Linux / macOS**, use the path as-is.

To detect which OS Blender is running on:
```python
import sys
is_windows = sys.platform == "win32"
glb_path = linux_to_windows_wsl(linux_path) if is_windows else linux_path
```

---

## Phase 5 — Import & Seat the Mesh

```python
import bpy, math, mathutils

def import_and_seat(glb_path, target_height=2.0, base_top_z=0.0):
    """
    Import HY3D GLB, scale to target_height, seat bottom at base_top_z.
    Returns list of imported mesh objects.
    """
    bpy.ops.object.select_all(action='DESELECT')
    bpy.ops.import_scene.gltf(filepath=glb_path)
    imported = [o for o in bpy.context.selected_objects if o.type == 'MESH']

    if not imported:
        raise RuntimeError("No mesh objects were imported. Check path and file.")

    # ── Measure current bounds ────────────────────────────────────────────────
    def get_bounds(objs):
        ax, ay, az = [], [], []
        for o in objs:
            for v in o.bound_box:
                wv = o.matrix_world @ mathutils.Vector(v)
                ax.append(wv.x); ay.append(wv.y); az.append(wv.z)
        return min(ax), max(ax), min(ay), max(ay), min(az), max(az)

    xn, xx, yn, yx, zn, zx = get_bounds(imported)
    height = zx - zn

    # ── Scale uniformly to target height ─────────────────────────────────────
    sf = target_height / height if height > 0 else 1.0
    for obj in imported:
        obj.name = "HY3D_" + obj.name
        obj.scale = (sf, sf, sf)
        bpy.ops.object.select_all(action='DESELECT')
        obj.select_set(True)
        bpy.context.view_layer.objects.active = obj
        bpy.ops.object.transform_apply(scale=True)

    # ── Re-measure after scale ────────────────────────────────────────────────
    xn, xx, yn, yx, zn, zx = get_bounds(imported)

    # ── Centre X/Y, seat bottom on base_top_z ────────────────────────────────
    cx = (xn + xx) / 2
    cy = (yn + yx) / 2
    for obj in imported:
        obj.location.x -= cx
        obj.location.y -= cy
        obj.location.z += (base_top_z - zn)

    # ── Shade smooth ──────────────────────────────────────────────────────────
    for obj in imported:
        bpy.ops.object.select_all(action='DESELECT')
        obj.select_set(True)
        bpy.context.view_layer.objects.active = obj
        bpy.ops.object.shade_smooth()

    # ── Verify final position ─────────────────────────────────────────────────
    xn2, xx2, yn2, yx2, zn2, zx2 = get_bounds(imported)
    print(f"[HY3D] Imported {len(imported)} objects | "
          f"height={zx2-zn2:.3f} | bottom_z={zn2:.3f} | "
          f"scale_factor={sf:.4f}")

    return imported
```

**Common import issues:**

| Problem | Symptom | Fix |
|---|---|---|
| Wrong OS path | `RuntimeError: Please select a file` | Convert Linux → Windows path (Phase 4) |
| Model 100× too large | Fills entire viewport | `sf = target_height / height` handles this automatically |
| Facing wrong direction | Dragon faces away from camera | `obj.rotation_euler.z = math.radians(N)` after import |
| Floating fragments | Small loose pieces floating | Select all → M → merge by distance, or delete tiny loose parts |
| Empty scene after import | `imported = []` | Check path exists; confirm WSL distro name (`Ubuntu` vs `Ubuntu-22.04`) |

---

## Phase 6 — Apply Material

Override any baked GLB material with a clean Principled BSDF:

```python
def apply_material(imported, name, r, g, b, roughness=0.5, metallic=0.0):
    mat = bpy.data.materials.new(name)
    mat.use_nodes = True
    bsdf = mat.node_tree.nodes["Principled BSDF"]
    bsdf.inputs["Base Color"].default_value    = (r, g, b, 1)
    bsdf.inputs["Roughness"].default_value     = roughness
    bsdf.inputs["Metallic"].default_value      = metallic
    bsdf.inputs["Specular IOR Level"].default_value = 0.15
    for obj in imported:
        obj.data.materials.clear()
        obj.data.materials.append(mat)

# Example: dark navy dragon
apply_material(imported, "Dragon_Dark_Navy", 0.06, 0.08, 0.18, roughness=0.48)
```

---

## Complete Workflow Checklist

```
[ ] 1. server_health()                         — confirm reachable
[ ] 2. generate_3d_async(text=... or image=...) — kick off generation
[ ] 3. Build base platform                     — disc + bevel only
[ ] 4. Build lighting rig                      — 3 area lights
[ ] 5. Add camera                              — set as scene camera
[ ] 6. check_generation_status(uid)            — poll until complete
[ ] 7. Convert path (Linux → Windows if needed)
[ ] 8. import_and_seat(glb_path, target_height, base_top_z)
[ ] 9. apply_material(imported, ...)
[ ] 10. Rotate model to face camera if needed
[ ] 11. Screenshot viewport to verify result
```

---

## What the Scene Should Contain

```
Scene
├── Base          ← cylinder disc, bevelled, stone/neutral material
├── Key_Light     ← area light, warm
├── Fill_Light    ← area light, cool
├── Rim_Light     ← area light, accent colour
├── Camera        ← set as scene.camera
└── HY3D_*        ← ONE imported mesh (the hero)
```

**Nothing else.** No procedural characters, no placeholder geometry, no duplicate models.
If the user requests additional props (e.g. a rock, a treasure chest), add them explicitly and only after the HY3D mesh is fully seated and verified.
