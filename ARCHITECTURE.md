# Architecture notes (scoped to what this fork touches)

Luanti is a large C++ voxel engine; this file covers only the client rendering
post-processing path, which is where the volumetric-perf branch lives. See
architecture.mmd for the diagram.

## Post-processing pipeline

`src/client/render/secondstage.cpp` — `addPostProcessing()` builds a chain of
`PostProcessingStep`s, each a fullscreen quad draw with a named shader from
`client/shaders/<name>/`. Steps read/write textures in a shared `TextureBuffer`
(indices TEXTURE_COLOR, TEXTURE_DEPTH, TEXTURE_BLOOM, TEXTURE_VOLUME, ...).

Order when bloom + volumetric are on:

1. 3D scene renders into TEXTURE_COLOR + TEXTURE_DEPTH (optionally via MSAA resolve).
2. `extract_bloom` → TEXTURE_BLOOM (bright-pass, full res).
3. `volumetric_light` reads (BLOOM, DEPTH) → TEXTURE_VOLUME. **Half res on this
   branch.** Carries the color through and adds sun/moon godrays raymarched
   against the depth buffer.
4. `bloom_downsample` ×4 mip chain (half, quarter, ... res) — source is
   TEXTURE_VOLUME when volumetric is on, else TEXTURE_BLOOM.
5. `bloom_upsample` back up the chain → TEXTURE_SCALE_UP.
6. Optional FXAA on TEXTURE_COLOR.
7. `second_stage` merges: sharp color (or FXAA) + blurred TEXTURE_SCALE_UP
   (bloom + godrays) + exposure → screen.

Key consequence: godrays are always blurred by the mip chain; their source
resolution barely matters. That is the load-bearing fact for this fork.

## The volumetric shader

`client/shaders/volumetric_light/opengl_fragment.glsl`:
- Screen-space raymarch from each pixel toward the sun/moon screen position,
  counting unoccluded (depth == 1, i.e. sky) samples along the ray; dithered
  with per-pixel noise.
- Result scaled by `pow(depth, 128)` (only sky-distance pixels glow), view-angle
  factors, and Rayleigh scattering color (`getDirectLightScatteringAtGround`).
- Uniforms (sun/moon screen pos, brightness, strength) fed from
  `src/client/game.cpp` / `src/client/shader.cpp` setter callbacks.
