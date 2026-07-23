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
cmake -B build -G Ninja -DCMAKE_BUILD_TYPE=Release -DENABLE_GETTEXT=ON \
      -DSDL2_DIR=/opt/homebrew/lib/cmake/SDL2 -DCMAKE_FIND_FRAMEWORK=LAST
ninja -C build
./bin/luanti          # runs from source tree, picks up local shaders
```

## Daily-driver .app (build-app/)

/Applications/luanti.app IS this fork (old release archived at
/Applications/luanti-5.16.1-release.app). Rebuild + reinstall:

```
ninja -C build-app install
codesign --force --deep -s - build-app/install/luanti.app   # REQUIRED: install rpath fixup invalidates the ad-hoc signature -> SIGKILL on launch
ditto build-app/install/luanti.app /Applications/luanti.app
```

Caveats: links against /opt/homebrew dylibs (this Mac only; a brew upgrade of
SDL2/luajit/etc. can break it — rebuild if the app dies on launch). Bundle
builds use ~/Library/Application Support/minetest for worlds/config, shared
with the old release app. The bundle's shaders are COPIES — shader edits need
a reinstall to reach the .app (unlike ./bin/luanti which reads the tree).

Upstream PR: https://github.com/luanti-org/luanti/pull/17360 (draft, branch
volumetric-light-perf = code changes only, pushed to fork ubuntuyou/luanti).

**SDL2 gotcha:** without the two SDL2 flags, CMake links against a stale
`~/Library/Frameworks/SDL2.framework` whose code signature macOS rejects at
load (`dyld: library load disallowed by system policy`). Always point it at
the Homebrew SDL2.

Deps via brew: cmake ninja freetype gettext gmp jpeg-turbo jsoncpp leveldb(optional)
libogg libpng libvorbis luajit sdl2 zstd.

## Testing volumetric perf

Enable in client: Settings → Graphics → Effects → Bloom + Volumetric lighting.
Compare F5 debug FPS vs the release app in the same world/time-of-day. Day + sun
on-screen = worst case (full raymarch); night/indoors should now be near-free.
