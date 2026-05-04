---
name: hunyuan3d-blender
description: "Use this skill when generating a 3D model from an image or text prompt via Hunyuan3D MCP tools and importing the result into Blender. Trigger whenever mcp__hunyuan3d tools are involved OR when the user says 'generate from image', 'generate from text', 'AI 3D model', or 'Hunyuan3D'. Always pair with blender-assembly skill when the imported mesh will be combined with other Blender geometry."
---

# Hunyuan3D → Blender Skill

Covers the full pipeline: generate a 3D mesh via Hunyuan3D MCP → import into Blender → integrate with scene geometry.

## Phase 1: Generate the Mesh

### Step 1 — Health check first
```python
server_health()   # confirm API is reachable before waiting on generation
```

### Step 2 — Choose model quality

| Use case | octree_resolution | num_inference_steps | notes |
|---|---|---|---
| Quick draft | 128 | 5 | turbo, fast |
| Final quality | 384 | 50 | slow (~2 min) |

### Step 3 — Generate

From image (preferred — more accurate geometry):
```python
path = generate_3d_from_image(
    image_path="/abs/path/to/image.png",
    seed=1234,
    octree_resolution=128,   # 128 draft / 384 quality
    num_inference_steps=5,   # 5 turbo / 50 quality
    guidance_scale=5.0,
)
# returns: absolute path to .glb file
```

From text:
```python
path = generate_3d_from_text(
    text="a wooden chair with four legs",
    seed=1234,
    octree_resolution=128,
    num_inference_steps=5,
    guidance_scale=5.0,
)
```

For long generations (quality mode), use async:
```python
uid = generate_3d_async(image_path="/abs/path/img.png", octree_resolution=384, num_inference_steps=50)
# poll every ~30s
result = check_generation_status(uid)   # returns path when done
```

Output is always a `.glb` file in `gradio_cache/`.

---

## Phase 2: Import into Blender

```python
import bpy, os

def import_hunyuan_glb(glb_path, name="HY3D_Mesh"):
    # Deselect all, import
    bpy.ops.object.select_all(action='DESELECT')
    bpy.ops.import_scene.gltf(filepath=glb_path)

    # Collect imported objects
    imported = [o for o in bpy.context.selected_objects]

    # Hunyuan3D exports in meters but scale factor varies — check Z extent
    for obj in imported:
        obj.name = name
        bpy.context.view_layer.objects.active = obj
        bpy.ops.object.transform_apply(location=False, rotation=True, scale=True)

    return imported
```

**Common import issues:**

| Problem | Fix |
|---|---|
| Model is 100× too large | `obj.scale = (0.01, 0.01, 0.01)` then `transform_apply(scale=True)` |
| Model faces wrong direction | Rotate 90° on X: `obj.rotation_euler.x = math.radians(90)` then `transform_apply` |
| Floating mesh fragments | Run `FloaterRemover` before export, or delete small loose parts in Blender |

Always call `verify_bounds(name)` (from blender-assembly) after import to check actual extents before positioning the mesh in scene.

---

## Phase 3: Integrate with Scene (use blender-assembly rules)

After import, treat the Hunyuan mesh like any other Blender object:

1. **`verify_bounds()`** on the imported mesh — get real extents
2. **Derive placement** from those bounds — never guess position relative to other parts
3. **`verify_overlap()`** at every joint between the AI mesh and hand-built geometry
4. **`finalize()`** before parenting or joining

```python
# Example: sit the AI mesh on a handbuilt platform
b_mesh  = verify_bounds("HY3D_Mesh")
b_plat  = verify_bounds("Platform")
# Place mesh so its bottom aligns with platform top
mesh_obj = bpy.data.objects["HY3D_Mesh"]
mesh_obj.location.z += b_plat['z'][1] - b_mesh['z'][0]
verify_overlap("HY3D_Mesh", "Platform", axis='z')
```

---

## Workflow Checklist

1. `server_health()` — confirm server up
2. Choose quality tier (128/5 draft or 384/50 final)
3. `generate_3d_from_image()` or `generate_3d_from_text()` — get `.glb` path
4. `import_hunyuan_glb()` — import + apply transforms
5. `verify_bounds()` — check actual scale/extents
6. Fix scale/orientation if needed, re-apply transforms
7. Use blender-assembly `verify_bounds` / `verify_overlap` / `finalize` for scene integration
