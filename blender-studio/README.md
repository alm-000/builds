# blender-studio

**Theme:** AI/automation · full-stack

*Concept / next build — no source code yet.*

## What I'm building

A Blender MCP (Model Context Protocol) pipeline that lets an AI model prompt, configure, and render 3D assets directly from natural language — without hand-animating scenes.

## The idea

Blender's Python API (`bpy`) exposes the full render pipeline programmatically. By wrapping it as an MCP server, an AI agent can:

1. Receive a natural-language prompt ("a low-poly mountain scene at dusk, isometric view").
2. Translate it into a sequence of `bpy` API calls: scene setup, material assignment, lighting, camera position, render settings.
3. Trigger a render and return the output image or video clip.

The goal is to make 3D asset generation as fast as text-to-image, but with deterministic geometry and full render control — useful for product mockups, social media assets, and repeatable branded visuals.

## Planned architecture

```
[Claude / AI agent]
       ↓ MCP tool calls
[blender-mcp-server]  ← Python, wraps bpy
       ↓
[Blender headless]    ← bpy + EEVEE/Cycles render
       ↓
[Output: PNG / MP4]
```

Key design choices:
- **Headless Blender** — runs without a GUI, scriptable via Python subprocess.
- **Template scenes** — base `.blend` files for common asset types (product shot, environment, character rig) so the agent modifies parameters rather than building from scratch.
- **Deterministic prompting** — structured MCP tool schema (not free-form text) keeps outputs reproducible.

## Status

Planning / early prototype. The MCP server skeleton and the `bpy` API layer are the first milestones.
