# PRD: GL_tile_shading Extension Support in glslang

## 1. Background

Tile-based deferred rendering (TBDR) architectures store intermediate rendering
results in fast on-chip tile memory. Today, access to this tile memory from
compute shaders is only available through vendor-specific extensions. There is a
clear need for a **vendor-neutral, open extension** that lets developers
explicitly control tile memory load/store operations on TBDR devices.

`glslang` already contains frontend support for vendor-specific tile shading
extensions, including stage validation, built-in handling, and test coverage.
This prior art serves as a useful implementation reference for bringing up the
proposed open extension `GL_tile_shading`.

## 2. Goal

Add non-production frontend support in `glslang` for `GL_tile_shading` with
enough coverage to compile prototype compute shaders that use
`tile_attachment_ext` images and tile-related built-ins, enabling early
validation and iteration on the extension design.

## 3. Non-goals

- Production-ready Vulkan extension support
- Final SPIR-V extension numbers or upstream-ready specification text
- Full driver/runtime validation

## 4. Target User Story

As a shader/compiler engineer, I want to compile a tile-shading compute shader
with `glslangValidator`, so that I can validate syntax changes, inspect
generated SPIR-V, and iterate on the extension design before formal
standardization.

## 5. API Usage Model

The extension enables interleaving graphics render passes with tile-shading
compute dispatches. Tile memory persists across passes within a command buffer,
avoiding costly round-trips to global memory.

```c++
vkCmdBeginCommandBuffer();

vkBeginRenderPass();                    // load tile
vkCmdBindPipeline(gfx_pipeline_1);
vkCmdDraw();                            // draw calls populate tile memory
vkEndRenderPass();                      // store tile

vkCmdBindPipeline(comp_pipeline_1);
VkCmdDispatch();                        // tile compute: load tile → process → store tile

vkBeginRenderPass();                    // load tile
vkCmdBindPipeline(gfx_pipeline_2);
vkCmdDraw();                            // further draw calls
vkEndRenderPass();                      // store tile

vkCmdBindPipeline(comp_pipeline_2);
VkCmdDispatch();                        // tile compute pass 2

vkCmdBindPipeline(comp_pipeline_3);
VkCmdDispatch();                        // tile compute pass 3

vkCmdEndCommandBuffer();
```

## 6. Shader Examples

The API sequence above maps to a minimal shader set:

- `gfx_pipeline_1` and `gfx_pipeline_2` are ordinary graphics pipelines that
  render into color/depth/stencil attachments in a render pass.
- `comp_pipeline_1`, `comp_pipeline_2`, and `comp_pipeline_3` are tile-shading
  compute passes that load those attachments as tile images, modify them, and
  store them back.
- `location = 0` maps to `color0` (`rgba32f`), `location = 1` maps to `color1`
  (`rgba32i`).
- `input0` is an auxiliary `rgba32f` tile image consumed only by
  `comp_pipeline_3`.
- Depth/stencil come from normal render-pass state; the compute pipelines can
  still read and update them through tile images.

### 6.1 Graphics Fragment Shader (pipeline 1)

```glsl
#version 460

layout(location = 0) in vec2 vUv;

layout(location = 0) out vec4 mrt0;
layout(location = 1) out ivec4 mrt1;

void main() {
    mrt0 = vec4(vUv, 0.25, 1.0);
    mrt1 = ivec4(int(vUv.x * 255.0),
                      int(vUv.y * 255.0),
                      1,
                      255);
    gl_FragDepth = mix(0.20, 0.80, vUv.x);
}
```

### 6.2 Tile Compute Shader (pipeline 1)

```glsl
#version 460
#extension GL_tile_shading : enable

layout(local_size_x = 16, local_size_y = 16, local_size_z = 1) in;

layout(set = 0, binding = 1, tile_attachment_ext, rgba32f)  uniform  image2D  color0;
layout(set = 0, binding = 2, tile_attachment_ext, rgba32i)  uniform iimage2D  color1;
layout(set = 0, binding = 3, tile_attachment_ext, rgba32f)  uniform  image2D  depth;
layout(set = 0, binding = 4, tile_attachment_ext, rgba32ui) uniform uimage2D  stencil;

layout(set = 0, binding = 5, rgba32ui) writeonly uimage2D tempout;

void main() {
    uvec2 tileMin = gl_TileOffsetEXT;
    uvec2 tileMax = tileMin + gl_TileDimensionEXT.xy - uvec2(1);
    ivec2 p = ivec2(clamp(gl_GlobalInvocationID.xy, tileMin, tileMax));

    vec4  c0 = imageLoadTileExt(color0, p);
    ivec4 c1 = imageLoadTileExt(color1, p);
    float  d = imageLoadTileExt(depth, p).x;
    uint   s = imageLoadTileExt(stencil, p).x;

    vec4 mixedColor = 0.5 * c0 + 0.5 * (vec4(c1) / 255.0);

    imageStoreTileExt(color0,  p, vec4(mixedColor.rgb, 1.0));
    imageStoreTileExt(color1,  p, c1 + ivec4(4, 0, 0, 0));
    imageStoreTileExt(depth,   p, vec4(clamp(d + 0.05, 0.0, 1.0)));
    imageStoreTileExt(stencil, p, uvec4(s + 1u));
    // write intermediate results to global memory (e.g. for HZB construction)
    imageStore(tempout, p, mixedColor);
}
```

### 6.3 Graphics Fragment Shader (pipeline 2)

```glsl
#version 460

layout(location = 0) in vec2 vUv;

layout(location = 0) out vec4 mrt0;
layout(location = 1) out ivec4 mrt1;

void main() {
    float checker = mod(floor(vUv.x * 8.0) + floor(vUv.y * 8.0), 2.0);

    mrt0 = mix(vec4(0.10, 0.20, 0.80, 1.0),
                    vec4(1.00, 0.70, 0.10, 1.0),
                    checker);
    mrt1 = ivec4(int(checker) * 32,
                      int((1.0 - checker) * 32.0),
                      2,
                      255);
    gl_FragDepth = mix(0.80, 0.20, vUv.y);
}
```

### 6.4 Tile Compute Shader (pipeline 2)

```glsl
#version 460
#extension GL_tile_shading : enable

layout(local_size_x = 16, local_size_y = 16, local_size_z = 1) in;

layout(set = 0, binding = 1, tile_attachment_ext, rgba32f)  uniform  image2D  color0;
layout(set = 0, binding = 2, tile_attachment_ext, rgba32i)  uniform iimage2D  color1;
layout(set = 0, binding = 3, tile_attachment_ext, rgba32f)  uniform  image2D  depth;
layout(set = 0, binding = 4, tile_attachment_ext, rgba32ui) uniform uimage2D  stencil;

void main() {
    uvec2 tileMin = gl_TileOffsetEXT;
    uvec2 tileMax = tileMin + gl_TileDimensionEXT.xy - uvec2(1);
    ivec2 p = ivec2(clamp(gl_GlobalInvocationID.xy, tileMin, tileMax));

    vec4  c0 = imageLoadTileExt(color0, p);
    ivec4 c1 = imageLoadTileExt(color1, p);
    float  d = imageLoadTileExt(depth, p).x;
    uint   s = imageLoadTileExt(stencil, p).x;

    float highlight = float((c1.x + c1.y) & 31) / 31.0;

    imageStoreTileExt(color0,  p, vec4(mix(c0.rgb, vec3(1.0), highlight * 0.25), 1.0));
    imageStoreTileExt(color1,  p, ivec4(c1.xy, 3, c1.w));
    imageStoreTileExt(depth,   p, vec4(max(0.0, d - 0.02)));
    imageStoreTileExt(stencil, p, uvec4(s ^ 1u));
}
```

### 6.5 Tile Compute Shader (pipeline 3)

```glsl
#version 460
#extension GL_tile_shading : enable

layout(local_size_x = 16, local_size_y = 16, local_size_z = 1) in;

layout(set = 0, binding = 0, tile_attachment_ext, rgba32f)  uniform  image2D  input0;
layout(set = 0, binding = 1, tile_attachment_ext, rgba32f)  uniform  image2D  color0;
layout(set = 0, binding = 2, tile_attachment_ext, rgba32i)  uniform iimage2D  color1;

void main() {
    uvec2 tileMin = gl_TileOffsetEXT;
    uvec2 tileMax = tileMin + gl_TileDimensionEXT.xy - uvec2(1);
    ivec2 p = ivec2(clamp(gl_GlobalInvocationID.xy, tileMin, tileMax));

    vec4  src = imageLoadTileExt(input0, p);
    vec4  c0  = imageLoadTileExt(color0, p);
    ivec4 c1  = imageLoadTileExt(color1, p);

    vec3 finalRgb = 0.6 * c0.rgb + 0.4 * src.rgb + vec3(c1.xyz) / 1024.0;

    imageStoreTileExt(color0, p, vec4(finalRgb, 1.0));
}
```

HLSL can mirror the same resource layout later; GLSL is the reference shape for
the initial frontend bring-up.

## 7. Functional Requirements

### FR-1 Extension Registration

- Add `GL_tile_shading` to extension tables in the GLSL frontend.
- Make `#extension GL_tile_shading : enable` and `: require` parse correctly.

### FR-2 Stage Restriction

- Restrict this extension to compute shaders in the initial version.
- Reject use from other stages with a clear diagnostic.

### FR-3 Layout Qualifier Support

- Accept `tile_attachment_ext` as a layout qualifier on image uniforms.
- Reuse existing image qualifier validation where possible.

### FR-4 Type and Access Model

- Support `image2D`, `iimage2D`, and `uimage2D` with `tile_attachment_ext`.
- Provide `imageLoadTileExt` and `imageStoreTileExt` built-in functions for
  tile image access.
- Preserve normal image format validation (`rgba32f`, `rgba32i`, `rgba32ui`).

### FR-5 Built-in Variables

- Define vendor-neutral built-in variables for tile shading:
  - `gl_TileOffsetEXT` — offset of the current tile in framebuffer coordinates.
  - `gl_TileDimensionEXT` — dimensions of the current tile.
- During initial bring-up, the implementation may map these to existing
  vendor-specific built-ins internally; this mapping should be documented in
  code comments and considered temporary.

### FR-6 SPIR-V Generation

- Generate valid SPIR-V for the accepted prototype shaders.
- For the initial branch, producing correct SPIR-V for prototype validation is
  the priority; upstream-ready SPIR-V extension plumbing is deferred.

## 8. Implementation Approach

### 8.1 Reference Path

Use the existing vendor-specific tile shading code paths in glslang as the
primary implementation reference, especially in:

- Parser and layout qualifier handling
- Stage validation
- Built-in symbol registration
- Tests under `Test/` and `gtests/`

### 8.2 Minimum Code Areas Expected to Change

- `glslang/MachineIndependent/Versions.h`
- `glslang/MachineIndependent/Versions.cpp`
- `glslang/MachineIndependent/Initialize.cpp`
- `glslang/MachineIndependent/ParseHelper.cpp`
- Parser files if a new layout token is needed
- Test inputs and baseline outputs

### 8.3 Rollout Phases

**Phase 1 — Syntax & Compilation**
- Extension registration
- Stage gating (compute only)
- `tile_attachment_ext` syntax acceptance
- Compile a minimal compute shader end-to-end

**Phase 2 — Built-ins & Validation**
- Built-in variable and function coverage for the prototype shaders
- SPIR-V inspection and stabilization
- Negative tests for invalid stage / invalid qualifier usage

**Phase 3 — Standardization Alignment**
- Finalize vendor-neutral naming for built-ins and SPIR-V mappings
- Evaluate alignment with or divergence from existing vendor extensions
- Prepare specification text for potential cross-vendor standardization

## 9. Validation and Acceptance

The feature is considered ready for this branch when all items below are true:

- `glslangValidator` accepts a minimal compute shader using
  `#extension GL_tile_shading : enable`
- `glslangValidator` rejects the extension in non-compute stages
- `tile_attachment_ext` images compile with valid formats and bindings
- At least one positive test shader is added under `Test/`
- At least one negative test is added for invalid stage or invalid qualifier use
- Generated SPIR-V is emitted successfully for the prototype shaders

## 10. Risks

- Parser support alone may hide missing SPIR-V or runtime semantics
- Upstreaming later will be harder if branch-only shortcuts are taken now
- Diverging too far from existing vendor implementations may complicate
  cross-vendor adoption

## 11. Open Questions

- Does `tile_attachment_ext` apply only to images, or also to attachment-like
  opaque types in later revisions?
- Do we need custom SPIR-V decorations, capabilities, or execution modes for
  the first usable branch?
- Is the target behavior "prototype compile only" or "stable internal branch
  used by application teams"?
- How should tile memory interact with frame-buffer compression formats (e.g.
  AFBC on ARM-based platforms)? Is this a shader-visible concern or purely a
  driver/hardware implementation detail?
