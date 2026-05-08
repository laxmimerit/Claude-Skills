---
name: character-image-to-3d
description: "Use this skill whenever generating, reconstructing, analyzing, or building a 3D character from one or more reference images inside Blender using MCP tools. Trigger this skill before ANY Blender character generation workflow — even if the user just says 'build this character', 'make this in 3D', 'create this model', or pastes a reference image. Covers stylized, anime, realistic, game-ready, cartoon, low-poly, high-poly, humanoid, creature, mascot, and cinematic characters from any source: screenshots, concept art, AI-generated images, sketches, or photos. This skill ensures correct proportion extraction, topology planning, staged construction, silhouette preservation, and animation-safe geometry. Do not skip this skill even for 'simple' characters."
---

# Character Image-to-3D Skill

Prevents the four biggest failures in AI-driven character modeling:
1. Wrong body proportions
2. Lost silhouette / character identity
3. Bad deformation topology
4. Overbuilding before structure is validated

**Run this skill before writing any Blender geometry code.**

---

## Pre-Flight Checklist

Before touching Blender, confirm all of these:

- [ ] Reference image analyzed (Phase 1)
- [ ] Reconstruction assumptions documented (Phase 2)
- [ ] Silhouette strategy defined (Phase 3)
- [ ] Construction strategy chosen — Low / Mid / High Poly (Phase 4)
- [ ] Primitive plan mapped per body part (Phase 6)
- [ ] MCP staging rules understood (Phase 10)

---

## Phase 1: Reference Image Analysis

Analyze the reference image and document:

```text
Character Analysis:
- Style: [Stylized Anime / Realistic / Cartoon / Game-ready / ...]
- Body Ratio: 1:[N] (head-to-body)
- Dominant Shape Language: [Rounded / Angular / Organic / ...]
- Clothing: [describe layers]
- Complexity: [Low / Medium / High]
- Rigging Difficulty: [Low / Medium / High]
- Hair Complexity: [Low / Medium / High]
- Accessories: [list]
- Dominant Colors: [list per zone]
```

**Never skip this phase. Never assume. Always document.**

---

## Phase 2: Multi-View Reconstruction Planning

One reference image hides information. Infer and document assumptions before modeling:

```text
Reconstruction Assumptions:
- Symmetry: [X-axis symmetrical / asymmetrical elements: ...]
- Hair volume: extends ~[N]cm behind skull
- Shoe sole thickness: ~[N]cm
- Backpack/accessory depth: ~[N]cm
- Hidden limb positions: [describe]
```

---

## Phase 3: Silhouette Preservation

**Silhouette is more important than surface detail.**

Viewers identify characters by outer shape, head profile, hairstyle, shoulder width, clothing mass, accessory placement — not by wrinkles, pores, or fabric stitching.

Lock the silhouette before adding any detail.

> **Rule: If the silhouette fails, the character fails.**

---

## Phase 4: Construction Strategy

Choose one strategy and document it before any geometry work:

| Strategy | Target | Notes |
|----------|--------|-------|
| Low Poly | Mobile, stylized, VR | <5k tris, flat shading acceptable |
| Mid Poly | Indie games, animated shorts | 5k–30k tris, smooth shading |
| High Poly | Sculpting, cinematic, baking | 30k+ tris, retopo required |

```text
Chosen Strategy: [Low / Mid / High Poly]
Modeling Style: [Box Modeling / Sculpt / Hybrid]
Retopology Required: [Yes / No]
```

---

## Phase 5: Construction Order

**Never build the full character at once.** Build modularly and validate proportions after each stage:

```
1. Torso blockout       ← screenshot + validate
2. Pelvis
3. Legs                 ← screenshot + validate
4. Arms                 ← screenshot + validate
5. Head                 ← screenshot + validate
6. Hair
7. Clothing             ← screenshot + validate
8. Accessories
9. Placeholder materials
10. Rig preparation     ← final screenshot all angles
```

---

## Phase 6: Primitive Plan

Map each body part to its starting primitive before generating geometry:

```python
# Head
bpy.ops.mesh.primitive_uv_sphere_add(segments=16, ring_count=12, radius=0.15)

# Torso
bpy.ops.mesh.primitive_cube_add(size=0.4)

# Arms / Legs
bpy.ops.mesh.primitive_cylinder_add(vertices=8, radius=0.04, depth=0.35)

# Eyes
bpy.ops.mesh.primitive_uv_sphere_add(segments=12, ring_count=8, radius=0.03)

# Hair — start as mesh, shape with proportional editing
bpy.ops.mesh.primitive_uv_sphere_add(segments=16, ring_count=12, radius=0.17)

# Backpack
bpy.ops.mesh.primitive_cube_add(size=0.25)
# then add bevel modifier
```

---

## Phase 7: Topology Rules

Animation-safe topology is mandatory.

**Required edge loops around:** eyes, mouth, shoulders, elbows, wrists, knees, ankles

**Avoid:**
- Triangles in deformation zones (joints, face)
- N-gons anywhere that deforms
- Stretched quads

**Maintain:**
- Evenly spaced quads in deformation areas
- Clean directional edge flow
- Symmetrical topology via mirror modifier

---

## Phase 8: Stylization Preservation

Do not drift from the source style. Common failure: stylized → accidental realism.

Actively preserve:
- Exaggerated eye size
- Oversized head relative to body
- Simplified cartoon hands
- Non-realistic limb proportions

> Stylization consistency > anatomical accuracy.

---

## Phase 9: Material Zones

Extract material zones before building shaders. Use flat placeholder materials first:

```python
# Assign placeholder material per zone
mat = bpy.data.materials.new(name="CHR_Hoodie_placeholder")
mat.diffuse_color = (0.2, 0.3, 0.8, 1.0)  # approximate color from reference
obj.data.materials.append(mat)
```

```text
Material Map (document before building):
- Hoodie    → matte fabric   (placeholder: flat blue)
- Boots     → rough leather  (placeholder: flat brown)
- Hair      → semi-gloss     (placeholder: flat black)
- Eyes      → glossy         (placeholder: flat white + iris sphere)
```

Do not build PBR or node shaders until geometry is finalized.

---

## Phase 10: MCP Execution Rules

| Rule | Detail |
|------|--------|
| **One stage per call** | Never generate full character in one `execute_blender_code` call |
| **Checkpoints** | `bpy.ops.wm.save_mainfile()` after every stage |
| **Validate before modifiers** | Use `get_object_detail_summary` to check mesh state first |
| **Apply transforms** | Run `transform_apply(location=True, rotation=True, scale=True)` after each stage |
| **Mirror early** | Add mirror modifier right after torso — model only one side |
| **Subdivision last** | No SubD modifiers until proportions are fully locked |
| **Naming convention** | `CHR_Body`, `CHR_Head`, `CHR_Hair_Base`, `CHR_LeftArm`, `CHR_Backpack` |

---

## Phase 11: Blender MCP Tool Reference

### Geometry creation and editing
Use `execute_blender_code` for all mesh ops, modifiers, and transforms:
```
Blender:execute_blender_code(code="...")
```

### Visual validation after each stage
```
Blender:get_screenshot_of_area_as_image(area_ui_type="VIEW_3D")
```
Call this after every major stage. Mentally black-fill the silhouette and compare to reference.

### Object state validation
```
Blender:get_object_detail_summary(name="CHR_Body")
```
Check transforms, modifiers, materials, and parent/child relationships before proceeding.

### Scene inventory check
```
Blender:get_blendfile_summary_datablocks()
```
Verify all expected objects exist with correct names before adding a new stage.

### Debug layout issues
```
Blender:get_screenshot_of_window_as_json()
```
Use when the viewport is missing or areas are misconfigured.

---

## Phase 12: Screenshot Validation Workflow

Required screenshot angles: **front · left · right · back · perspective**

```python
# Switch to front ortho view before screenshotting
import bpy
for area in bpy.context.screen.areas:
    if area.type == 'VIEW_3D':
        for space in area.spaces:
            if space.type == 'VIEW_3D':
                space.region_3d.view_perspective = 'ORTHO'
        with bpy.context.temp_override(area=area):
            bpy.ops.view3d.view_axis(type='FRONT')
```

Then call `get_screenshot_of_area_as_image`. Repeat for LEFT, RIGHT, BACK. Switch to PERSP for the perspective shot.

**Silhouette check rule:** If the shape reads differently from the reference when mentally filled solid black, fix it before continuing.

---

## Phase 13: Rigging Readiness

Before finalizing, verify:

```python
# Ensure origin is at world center
bpy.ops.object.origin_set(type='ORIGIN_GEOMETRY', center='BOUNDS')
bpy.context.object.location = (0, 0, 0)

# Confirm pose is T-pose (arms extended, palms down) or A-pose (arms 45° down)
# Legs straight, feet flat on ground plane
```

Rigging checklist:
- [ ] T-pose or A-pose
- [ ] Origin at scene center (0, 0, 0)
- [ ] Clean X-axis symmetry
- [ ] Edge loops at all joints (shoulders, elbows, wrists, hips, knees, ankles)
- [ ] Fingers separated (not merged)
- [ ] Mouth loop clean and closed

---

## Common Failures and Fixes

| Failure | Cause | Fix |
|---------|-------|-----|
| Looks different from reference | Silhouette ignored | Black-fill outline, compare shape |
| Deforms badly at joints | Missing edge loops | Rebuild loops around joint |
| Proportions feel wrong | No head/body ratio check | Measure head height vs total height |
| Character feels lifeless | Detail before primary forms | Strip to blockout, revalidate |
| Stylized → accidentally realistic | No stylization check | Exaggerate key proportions deliberately |
| Modifier causes mesh explosion | Transform not applied | `transform_apply()` before every modifier |
| Objects misnamed or lost | No naming discipline | Check with `get_blendfile_summary_datablocks()` |