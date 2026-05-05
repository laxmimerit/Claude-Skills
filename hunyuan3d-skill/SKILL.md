---
name: hunyuan3d-blender
description: "Use this skill when generating a 3D model from an image or text prompt via Hunyuan3D MCP tools and importing the result into Blender. Trigger whenever mcp__hunyuan3d tools are involved OR when the user says 'generate from image', 'generate from text', 'AI 3D model', or 'Hunyuan3D'."
---

# Hunyuan3D → Blender Skill

## Pipeline Order

1. `server_health` — stop if unreachable
2. `generate_3d_async` — kick off generation immediately
3. Build scene while waiting — base + lights + camera only
4. `check_generation_status` — poll until complete
5. Convert path if on Windows
6. Import, scale, seat, shade smooth
7. Apply material
8. Screenshot to verify

---

## Critical Rules

- **The HY3D mesh is the only hero object.** Never build a procedural version of the requested model while waiting — not even as a placeholder.
- **One hero per scene** unless the user explicitly requests multiple.
- **Never auto-clear the scene.** If the scene needs clearing, ask the user first.
- **Never guess scale or position.** Always measure bounds after import and derive placement from real numbers.
- **Never use raw Linux paths in Blender on Windows.** Convert first — see Path Conversion below.

---

## While Waiting — Scene Setup

Build exactly three things, nothing else: a base disc, a 3-point lighting rig, and a camera set as `scene.camera`.

**Base:** bevelled cylinder, radius ~1.4, neutral/stone material, shade smooth.

**Lighting:** 3 area lights — warm key from front-right, cool fill from front-left, accent rim from behind. Tune color and energy to match the character's mood.

**Camera:** 85mm lens, 3/4 hero angle looking down at the model. Set as `scene.camera`.

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

## Import & Seat Rules

- Deselect using `obj.select_set(False)` on all objects — never `bpy.ops.object.select_all` which crashes outside viewport context.
- After import, collect only `MESH` type objects from `bpy.context.selected_objects`.
- If the result is empty, stop and raise a clear error — do not proceed silently.
- Rename all imported objects with `HY3D_` prefix.
- Scale uniformly so the model fits `target_height=2.0` units. Derive scale factor from measured Z bounds, never guess.
- Re-measure bounds after scaling. Centre on X/Y. Seat bottom at `z=0` (top of base disc).
- Shade smooth after seating.
- After seating, check model orientation. If it faces away from camera, rotate on Z axis to face front.
- Print final height, bottom_z, and scale factor to confirm correctness.

---

## Material Rules

- Always override baked GLB materials with a fresh Principled BSDF.
- Match base color and roughness to the character's intended look.

---

## Expected Final Scene

```
Scene
├── Base          ← bevelled disc, neutral material
├── Key_Light     ← area, warm
├── Fill_Light    ← area, cool
├── Rim_Light     ← area, accent
├── Camera        ← scene.camera, 85mm
└── HY3D_*        ← ONE mesh, bottom at z=0
```

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| `RuntimeError: Please select a file` | Missing path conversion — run `linux_to_windows_wsl()` |
| `imported = []` | Wrong WSL distro name, or file not yet written |
| `poll() failed, context is incorrect` | Replace `bpy.ops.object.select_all` with `obj.select_set(False)` loop |
| Model too large or tiny | Re-derive scale factor from actual Z bounds |
| Model faces wrong way | Rotate on Z axis after import |
| Floating mesh fragments | Edit Mode → Merge by Distance |
