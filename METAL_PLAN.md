# Metal Rendering Path Optimization Plan

## Problem Summary

From two Metal HUD captures (5120×2688 @ 2.0 scale, ~23 FPS):

- **495 compute encoders/frame** — most do 0.00–0.01ms GPU work (pure overhead)
- **60 render encoders/frame** — first large one ~2.4ms, then 59 tiny ones (0.01–0.09ms)
- **4 blit encoders/frame** — CPU time 34–37ms avg, spikes to 146–158ms
- **~16,301 "Clear Image" operations** over capture (render-pass-based)
- **Fragment GPU: 14–16ms** — dominant GPU cost
- **Performance Insight**: "Frequent Render Target Change — render passes are expensive on Apple GPUs"

The GPU is bottlenecked by `endEncoding()`/`beginEncoding()` overhead rather than actual work.

---

## Fix 1: Compute Encoder Reuse (HIGH IMPACT, LOW RISK)

**Files:** `drivers/metal/metal3_objects.cpp`, `drivers/metal/metal3_objects.h`

**Root cause:** `bind_pipeline()` (line 221–222) calls `_end_compute_dispatch()` when binding a new compute pipeline, destroying the current `MTLComputeCommandEncoder`. Then `_compute_set_dirty_state()` (line 1380) creates a brand new encoder on the next dispatch. With 495 compute dispatches using different pipelines, this creates 495 separate compute encoders.

### Change 1a: Don't end compute encoder on pipeline change

In `bind_pipeline()`, replace:

```cpp
// Old: destroys encoder on any compute pipeline change
if (type == MDCommandBufferStateType::Compute) {
    _end_compute_dispatch();
} else if (type == MDCommandBufferStateType::Blit) {
    _end_blit();
}
```

With:

```cpp
// New: only destroy when transitioning AWAY from compute (to render)
if (p->type == MDPipelineType::Render) {
    if (type == MDCommandBufferStateType::Compute) {
        _end_compute_dispatch();
    } else if (type == MDCommandBufferStateType::Blit) {
        _end_blit();
    }
    // ... render pipeline binding unchanged ...
} else if (p->type == MDPipelineType::Compute) {
    if (type == MDCommandBufferStateType::Blit) {
        _end_blit();
    }
    if (type != MDCommandBufferStateType::Compute) {
        type = MDCommandBufferStateType::Compute;
    }
    if (compute.pipeline != p) {
        compute.dirty.set_flag(ComputeState::DIRTY_PIPELINE);
        binding_cache.clear();
        compute.mark_uniforms_dirty();
        compute.pipeline = (MDComputePipeline *)p;
    }
}
```

### Change 1b: Reuse existing compute encoder

In `_compute_set_dirty_state()`, replace:

```cpp
if (compute.dirty.has_flag(ComputeState::DIRTY_PIPELINE)) {
    compute.encoder = NS::RetainPtr(command_buffer()->computeCommandEncoder(MTL::DispatchTypeConcurrent));
    _encode_barrier(compute.encoder.get());
    compute.encoder->setComputePipelineState(compute.pipeline->state.get());
}
```

With:

```cpp
if (compute.dirty.has_flag(ComputeState::DIRTY_PIPELINE)) {
    if (compute.encoder.get() == nullptr) {
        compute.encoder = NS::RetainPtr(command_buffer()->computeCommandEncoder(MTL::DispatchTypeConcurrent));
        _encode_barrier(compute.encoder.get());
    }
    compute.encoder->setComputePipelineState(compute.pipeline->state.get());
}
```

**Impact:** Reduces compute encoders from 495 to 1 per frame. Eliminates 494 endEncoding/beginEncoding pairs.

---

## Fix 2: Compute Shader Texture Clearing (HIGH IMPACT, MODERATE RISK)

**Files:** `drivers/metal/metal3_objects.cpp` + new `drivers/metal/metal_clear_kernel.h`

**Root cause:** `clear_color_texture()` (line 329–404) iterates mip levels and layers, creating one `MTLRenderCommandEncoder` per mip (and per layer when layered rendering unavailable). ~139 clears/frame × multiple encoders each = significant overhead.

### Change 2a: Add inline clear compute kernel

Embed Metal shader source for clearing textures via compute:

```metal
kernel void clear_texture_2d(
    texture2d<float, access::write> tex [[texture(0)]],
    constant float4& color [[buffer(0)]],
    uint2 gid [[thread_position_in_grid]])
{
    tex.write(color, gid);
}
```

### Change 2b: Rewrite `clear_color_texture()`

Instead of per-mip render encoders, use a compute pipeline:
- Lazily create a `MTLComputePipelineState` for clear kernels
- For each mip level, create a texture view, bind it, dispatch
- One dispatch per mip level, zero render encoder creation

### Change 2c: Reduce unnecessary clears

Sources of clears (all in `servers/rendering/renderer_rd/`):
- `gi.cpp`: lightprobe history/average/emission
- `ss_effects.cpp`: last-frame texture, SDF
- `fog.cpp`: density/light/emissive maps
- `fsr2.cpp`: FSR2 resources
- `texture_storage.cpp`: atlas/decal atlas

Audit whether any clear every frame is needed vs. only on first use/resize.

---

## Fix 3: Render Pass Merging (MEDIUM IMPACT, HIGHER RISK)

**File:** `drivers/metal/metal3_objects.cpp`

**Root cause:** Performance Insight: "Frequent Render Target Change". Each draw list in the draw graph creates a separate render pass with its own encoder.

### Change 3a: Subpass merging

In `render_next_subpass()`, when consecutive subpasses share the same attachments, avoid `_end_render_pass()` + new encoder. Reuse the existing encoder.

### Change 3b: Defer blit operations

`_ensure_blit_encoder()` (line 288) ends any active render/compute encoder before creating a blit encoder. Buffer blit operations and apply them at natural type boundaries.

---

## Expected Improvement

| Metric | Before | After (estimated) |
|--------|--------|-------------------|
| Compute encoders/frame | 495 | 1 |
| Render encoders/frame | 60 | ~50 |
| Encoder CPU time | 36–40ms | ~1–5ms |
| Peak CPU time | 155ms | ~20–30ms |
| Average frame time | 42ms (24fps) | ~20ms (50fps) |

---

## Status: 2026-08-08 capture (after Fix 1 + Fix 2 landed)

Fixes 1 and 2 are merged into HEAD (`5513643b27` compute encoder reuse,
`2f881db84c` compute texture clear). New capture
`libMTLHud_Godot_2026_08_08_05_03_26_5s.html` @5120x2690, scale 2.0.

### Results vs. stock 4.7 stable

| Metric (average) | Stock 4.7 | Best prior (Jun 22:11) | Aug 8 capture |
|---|---|---|---|
| Frame interval | 45.3ms | 26.3ms | 28.9ms |
| Command Buffer GPU | 48.4ms | 29.4ms | 32.0ms |
| Encoder CPU | 39.9ms | 24.5ms | 24.6ms |
| Compute encoders/frame | 495 | 39 | 40 |
| Compute GPU | 16.0ms | 11.3ms | 11.2ms |
| Render encoders/frame | 60 | 51 | 59 |
| Fragment GPU | 15.8ms | 13.9ms | 16.7ms |
| Blit Encoder CPU | 37.4ms | 22.3ms | 22.1ms |
| Blit GPU | 0.51ms | 0.73ms | 0.53ms |
| "Frequent RT Change" insight | present | gone | gone |

### What's holding

- Compute encoders 495 → 40 (-92%), encoder CPU 39.9 → 24.6ms (-38%).
- Frame interval 45.3 → 28.9ms (-36%). RT-change insight gone.
- Fixed-frame budget is 33.3ms (30Hz); frame interval 28.9ms is within it.

### Remaining targets (in priority order)

1. **Blit Encoder CPU ~22ms/frame avg, spikes to 73ms** — now the dominant
   CPU cost and the biggest remaining win. `_ensure_blit_encoder()`
   (`metal3_objects.cpp:297`) ends any active render/compute pass and emits a
   barrier around every blit. Frame 2941 shows `Blit Encoder 2` at 60ms CPU /
   0.05ms GPU — pure churn. This is the unimplemented **Fix 3b (defer blit
   operations)**. Buffer/texture copy ops (`clear_buffer`, `copy_buffer`,
   `copy_texture`) should batch into one blit encoder applied at natural
   type boundaries instead of opening/ending one per op.
2. **Fragment GPU 16.7ms** — GPU side is now the other half of the bottleneck
   (Command Buffer GPU 32.0ms of 33.3ms budget). Needs investigation into
   shader/fragment cost (FXAA, SSAO, tonemap, etc.); could benefit from
   resolution or effect reduction.
3. **Render encoders 59 (vs 51 best)** — depth/stencil clears still use
   per-mip render passes (`clear_depth_stencil_texture`,
   `metal3_objects.cpp:536`). The compute-clear change only converted *color*
   clears. `MTL::BlitCommandEncoder` can't clear depth/stencil to arbitrary
   values either, so this needs a compute kernel writing depth texture views
   or staying on render clears; lower priority than blit batching.

### Notes / pitfalls

- The HUD's Metrics table reports FPS/Frame Interval as garbage
  (`9223372036854775807`, `0.00ms`) — a HUD formatting quirk. Trust the
  **Frame Interval Distribution** table (28.90ms avg, 0 missed) instead.
- Render Encoder CPU is ~1.8ms, Render Encoder GPU ~27.6ms — encoder GPU is
  dominated by fragment cost, not encoder count.
- Compare only same-resolution captures (5120x2688/2690 @ scale 2.0); the
  June 09:02 and stock-master captures at the same res are the baseline set.
