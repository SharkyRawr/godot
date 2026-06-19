# Metal Rendering Device: Compute Encoder Reuse & Compute-Based Texture Clearing

## Summary

Two optimizations to the Metal rendering device driver that eliminate excessive command encoder creation, which was the primary CPU and GPU bottleneck on Apple's tile-based deferred rendering (TBDR) GPUs.

- **Stage 1:** Reuse a single compute command encoder across pipeline changes instead of destroying and recreating it for every dispatch.
- **Stage 2:** Replace per-mip render-pass-based texture clears with compute kernel dispatches, eliminating the creation of per-mip render command encoders entirely.

## Problem

Metal HUD profiling of the editor at 5120×2888 revealed that the GPU was bottlenecked by encoder overhead rather than actual rendering work:

- **495 compute encoders per frame**, each doing 0.00–0.01ms of GPU work. Every `endEncoding()`/`beginEncoding()` pair on a TBDR GPU forces a tile store/load cycle through memory — pure overhead.
- **60 render encoders per frame**, with ~9 of them being "Clear Image" passes created solely to clear texture subresources. Each clear of a multi-mip texture created one render encoder per mip level per layer.
- **~16,301 "Clear Image" render passes** across a capture, flagged by Metal's "Frequent Render Target Change" performance insight.
- **Result:** 18 FPS, 56ms frame intervals, with the GPU spending more time on encoder transitions than on actual fragment or compute work.

## Changes

### Stage 1: Compute Encoder Reuse

**Files:** `drivers/metal/metal3_objects.cpp`, `drivers/metal/metal3_objects.h`

Previously, `bind_pipeline()` called `_end_compute_dispatch()` whenever a new compute pipeline was bound, destroying the current `MTLComputeCommandEncoder`. The next dispatch then created a brand new encoder via `_compute_set_dirty_state()`. With 495 dispatches using different pipelines, this created 495 separate encoders per frame.

**Fix:** When binding a new compute pipeline while already in compute mode, keep the existing encoder alive. Only call `setComputePipelineState()` on the existing encoder. The encoder is now created once and reused for all subsequent compute dispatches within the frame.

The encoder is still ended when transitioning to a different encoder type (render or blit), preserving correct execution ordering.

### Stage 2: Compute-Based Texture Clearing

**Files:** `drivers/metal/metal3_objects.cpp`, `drivers/metal/metal3_objects.h`, `drivers/metal/metal_objects_shared.cpp`, `drivers/metal/metal_objects_shared.h`, `drivers/metal/rendering_device_driver_metal.cpp`

`clear_color_texture()` previously created one `MTLRenderCommandEncoder` per mip level (and per layer when layered rendering was unavailable). Each encoder performed a render pass with `LoadActionClear` and immediately ended — no actual drawing occurred.

**Fix:** Replaced the render-pass-based clear with compute kernel dispatches:

1. Three MSL compute kernels (`clear_color_2d`, `clear_color_2d_array`, `clear_color_3d`) handle the respective texture types, writing a uniform color to every texel via `access::write` textures.

2. Pipeline states for these kernels are lazily compiled and cached in `MDResourceCache`, one per texture type.

3. `MDResourceFactory::new_clear_color_compute_pipeline_state()` compiles the kernel source on first use and returns a reusable `MTLComputePipelineState`.

4. `MDCommandBuffer::_clear_color_texture_compute()` creates a single compute encoder, binds the cached pipeline, and dispatches once per mip level using texture views scoped to that level. All mips are cleared within one encoder — no per-mip encoder creation.

5. Textures created with `CAN_UPDATE_BIT | CAN_COPY_TO_BIT` and color attachment capability now also get `MTL::TextureUsageShaderWrite` set (excluding multisample textures, which cannot be written from compute kernels). This allows the compute clear path to be used.

6. Multisample textures, cube textures, and other unsupported types fall back to the original render-pass-based clear path automatically.

7. `dispatchThreads` is used instead of `dispatchThreadgroups` so Metal handles the threadgroup count calculation automatically — the grid size is simply the texture dimensions at each mip level.

8. The compute clear path is only used for float-compatible formats (`ColorFloat`/`ColorHalf` via `PixelFormats::getFormatType()`). Integer formats, multisample textures, cube textures, and other unsupported types fall back to the original render-pass-based clear path automatically.

**Threadgroup sizes:** 8×8×1 for 2D and 2D array textures, 8×8×4 for 3D textures.

## Design Decisions

### Why compute instead of render passes for clearing?

On Apple's TBDR GPUs, each render pass encoder forces the tile memory to be flushed to system memory at `endEncoding()` and reloaded at the next `beginEncoding()`. For a clear operation that writes one color value to every texel, this store/load cycle costs more than the clear itself. A compute encoder writing to a `texture2d<float, access::write>` stays in the compute pipeline and doesn't trigger tile flushes the same way.

### Why not use `MTL::BlitCommandEncoder` for clears?

Metal's blit encoder does not support clearing color textures to an arbitrary color value — it only supports clearing to zero via `fillBuffer`. Render passes and compute kernels are the only options.

### Why set `ShaderWrite` at texture creation time?

The compute clear path requires the texture to have `MTL::TextureUsageShaderWrite` in its usage flags. Rather than retroactively checking or modifying usage at clear time, we set the flag at creation for textures that are clearable color attachments. On Apple GPUs there is no performance penalty for having this flag set — it only affects how the GPU allocates internal resources, and Apple's unified memory architecture doesn't distinguish between shader-read and shader-write textures the way discrete GPUs do.

The flag is only set when the format's capability table reports both `kMTLFmtCapsWrite` and `kMTLFmtCapsColorAtt`, and when the format type is `ColorFloat` or `ColorHalf` (covering unorm, snorm, sRGB, and float). This correctly excludes:
- **Integer formats** (`ColorUInt*`, `ColorInt*`), whose compute write semantics differ from the float clear kernel.
- **sRGB formats on simulator**, where the capability table clears `kMTLFmtCapsWrite`.

### Why gate the compute clear path on format type?

The MSL clear kernels use `texture*<float, access::write>`, which is only valid for float, half, unorm, and snorm formats (all classified as `MTLFormatType::ColorFloat` or `ColorHalf`). Integer formats (`ColorUInt*`, `ColorInt*`) require `texture*<uint/int, access::write>` kernels with different write semantics. Rather than maintaining separate int/uint kernels, we check `PixelFormats::getFormatType()` at clear time and fall back to the render clear path for integer formats. This keeps the optimization targeted at the common case (float/unorm color textures) without risking validation failures on integer textures created by modules like Betsy.

### Why not merge all clears into one dispatch?

Each mip level has different dimensions, so the dispatch grid size differs per mip. Texture views are used to isolate a single mip level for the write. Attempting to clear all mips in a single dispatch would require a more complex kernel with indirect dispatch arguments, and the savings would be marginal since the encoder itself is already shared across all mips.

### Why exclude multisample textures?

Metal compute kernels cannot write to multisample textures (`texture2d_ms`) via `access::write`. Multisample textures also cannot be used as `texture2d` in compute. The render-pass clear path is retained as a fallback for these and other unsupported texture types (cube, 1D).

## Performance Results

Measured via Metal HUD, 5-second capture (~130 frames), same editor scene, 5120×2888 resolution.

| Metric | Before (stock) | After | Change |
|---|---|---|---|
| **FPS (average)** | 18 | 27 | **+50%** |
| **FPS (last frame)** | 20 | 30 | **+50%** |
| **Frame Interval (average)** | 56.00ms | 38.89ms | **-31%** |
| **Frame Interval (last)** | 50.00ms | 33.34ms | **-33%** |
| **Compute Encoder Count** | 495 | 39 | **-92%** |
| **Render Encoder Count** | 60 | 51 | **-15%** |
| **"Clear Image" passes** | 1,818 | 0 | **eliminated*** |
| **"Frequent RT Change" insight** | present | gone | **eliminated** |
| **Fragment GPU (avg)** | 21.87ms | 19.34ms | **-12%** |
| **Fragment GPU (last)** | 33.29ms | 18.81ms | **-43%** |
| **Compute GPU (avg)** | 22.36ms | 18.47ms | **-17%** |
| **Command Buffer GPU (avg)** | 59.82ms | 42.30ms | **-29%** |

The compute encoder count dropped from 495 to 39 (stage 1), and all measured clear-image passes in this scene were eliminated (stage 2). Eligible float-compatible 2D/2D-array/3D clears use compute; fallback clears (integer formats, multisample, cube) may still emit render passes. The remaining 39 compute encoders and 51 render encoders represent actual rendering work from Godot's draw graph — distinct render targets and compute passes that cannot be merged at the driver level.

\* No "Clear Image" labeled encoders appeared in the tested scene. Textures that fall back to the render clear path (integer formats, multisample, unsupported texture types) would still emit them if cleared.

## Files Changed

| File | Lines | Description |
|---|---|---|
| `drivers/metal/metal3_objects.cpp` | +164 / -53 | Compute encoder reuse in `bind_pipeline`/`_compute_set_dirty_state`; compute-based `_clear_color_texture_compute` with format-type gating and mip prevalidation; render fallback `_clear_color_texture_render` |
| `drivers/metal/metal3_objects.h` | +2 | Private method declarations |
| `drivers/metal/metal_objects_shared.cpp` | +80 | `MDResourceFactory::new_clear_color_compute_pipeline_state` with embedded MSL kernels; `MDResourceCache::get_clear_color_compute_pipeline_state` cache accessor |
| `drivers/metal/metal_objects_shared.h` | +6 | Cache pipeline state members and method declarations |
| `drivers/metal/rendering_device_driver_metal.cpp` | +11 | `TextureUsageShaderWrite` flag gated on `kMTLFmtCapsWrite`, `kMTLFmtCapsColorAtt`, and float-compatible format type for clearable color textures |

**Total:** 5 files, +263 / -53 lines
