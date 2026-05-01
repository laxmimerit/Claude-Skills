---
name: hunyuan3d-skill
description: Interface with Hunyuan3D 2.0 via MCP tools to generate high-resolution textured 3D assets. Use when the user wants to trigger 3D synthesis (Text-to-3D or Image-to-3D) using Model Context Protocol (MCP) tools.
---

# Hunyuan3D 2.0 MCP Skill

## Overview
This skill guides the use of Hunyuan3D 2.0 through MCP (Model Context Protocol) tools. It focuses on the agentic workflow of transforming text prompts or images into production-ready 3D meshes (GLB/OBJ).

## MCP Tool Workflow

When this skill is active, you should look for and utilize the following types of MCP tools (names may vary based on your local MCP server configuration):

### 1. `generate_3d_from_text`
- **When to use**: To create a 3D model from a text description.
- **Inputs**: 
    - `prompt`: Detailed description of the object.
    - `steps` (optional): Number of diffusion steps (default 30-50).
- **Strategy**: Encourage the user to provide architectural or material details for better results.

### 2. `generate_3d_from_image`
- **When to use**: To create a 3D model from a reference image.
- **Inputs**: 
    - `image_path` or `image_data`: Path to the local image or base64 data.
    - `texture_gen` (optional): Boolean to enable/disable the Hunyuan3D-Paint stage.
- **Strategy**: If the image has a complex background, recommend using a background removal tool first for cleaner geometry.

### 3. `apply_3d_texture`
- **When to use**: To apply high-resolution textures to an existing mesh using a reference image.
- **Inputs**:
    - `mesh_path`: Path to the untextured `.obj` or `.glb`.
    - `reference_image`: Image used to guide the texture synthesis.

## Technical Context for Agents
Hunyuan3D 2.0 operates in two distinct stages which the MCP tools abstract:
1.  **Stage 1: Shape Generation (Hunyuan3D-DiT)** - A flow-based diffusion transformer creates the initial geometry.
2.  **Stage 2: Texture Synthesis (Hunyuan3D-Paint)** - A 1.3B parameter model applies 4K textures and PBR materials.

## Best Practices
- **Resolution**: Use the highest available resolution for input images (Stage 1 benefits from clear silhouettes).
- **VRAM**: If the MCP server reports "Out of Memory," suggest using the "mini" model variant if supported by the tool parameters.
- **Feedback**: After generation, provide the user with the path to the exported `.glb` and suggest opening it in a 3D viewer (like Blender) for inspection.

## Error Handling
- **Missing Weights**: If the tool fails with a "Weights not found" error, advise the user to check their `tencent/Hunyuan3D-2` download on Hugging Face.
- **Connection Error**: If the MCP server is unreachable, ensure `api_server.py` is running in the Hunyuan3D-2 environment.
