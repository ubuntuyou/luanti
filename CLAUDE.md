# Luanti (volumetric-perf fork)

Shallow clone of luanti-org/luanti master (c85db93, ~5.16-dev). Branch `volumetric-perf`
carries local perf changes to the volumetric lighting (godrays) pass. Installed release
client for A/B comparison: /Applications/luanti.app (5.16.1).

## Gotchas / invariants

- **Volumetric light only reaches the screen through the bloom blur chain.** The
  `volumetric_light` shader writes TEXTURE_VOLUME, which is consumed solely by the
  bloom downsample→upsample chain, then merged in `second_stage` via TEXTURE_SCALE_UP.
  This is why rendering it below full res is nearly free visually. If upstream ever
  composites TEXTURE_VOLUME directly, revisit the half-res change.
- Volumetric requires bloom: `enable_volumetric_lighting` is gated on `enable_bloom`
  (secondstage.cpp).
- Shaders live in `client/shaders/<name>/opengl_fragment.glsl` and are loaded at
  runtime from the source tree — shader edits need no recompile, just restart the
  client (or rejoin). C++ pipeline changes need a rebuild.
- `pow(depth, 128.)` in the shader means only sky-distance pixels contribute godrays;
  the early-outs depend on this.
- Depth texture stays NEAREST-filtered in the volumetric step: the shader does exact
  `< 1.` comparisons on depth.

## Local changes (branch volumetric-perf)

1. TEXTURE_VOLUME at `scale * 0.5f` + bilinear on color input — secondstage.cpp
2. Shader: skip raymarch when `brightness <= 0` (sun/moon not visible)
3. Shader: skip raymarch when `pow(rawDepth,128) < 1e-4` (near geometry)
4. Shader: 30 → 16 samples; reuse `rawDepth` instead of re-sampling depthmap

## Build

```
cmake -B build -G Ninja -DCMAKE_BUILD_TYPE=Release -DENABLE_GETTEXT=ON
ninja -C build
./bin/luanti          # runs from source tree, picks up local shaders
```

Deps via brew: cmake ninja freetype gettext gmp jpeg-turbo jsoncpp leveldb(optional)
libogg libpng libvorbis luajit sdl2 zstd.

## Testing volumetric perf

Enable in client: Settings → Graphics → Effects → Bloom + Volumetric lighting.
Compare F5 debug FPS vs the release app in the same world/time-of-day. Day + sun
on-screen = worst case (full raymarch); night/indoors should now be near-free.
