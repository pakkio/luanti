# Optimization ideas for `irr/` (Luanti's Irrlicht fork)

Status: **code review only.** Nothing has been built or profiled yet, so every
impact estimate below is a hypothesis. Measure before changing anything.

## Hot path

World rendering: `src/client/clientmap.cpp` (draw loop, ~line 1144)
-> `setMaterial` (only on material change) -> `setTransform` -> `descriptor.draw()`
-> `COpenGL3DriverBase::drawBuffers` (`irr/src/OpenGL/Driver.cpp:573`).

Map chunks are already merged into larger buffers (`transformBuffersToDrawOrder`),
so draw call counts are lower than a naive per-block count. Entities, particles
and other non-map meshes are likely where per-draw driver overhead matters most.

## Ideas, ranked by my guess of impact

### 1. Vertex array objects (VAOs)
- **Where:** `Driver.cpp:573` (`drawBuffers`), `:1046` (`beginDraw`), `:1069` (`endDraw`).
- **Problem:** every draw binds buffers, re-specifies every vertex attribute
  (enable + pointer), draws, then disables every attribute and unbinds.
  Roughly 15-20 GL calls of overhead per draw.
- **Fix:** cache a VAO per (vertex buffer, index buffer, vertex type) and bind
  one object per draw. GL3 / GLES3 have VAOs; GLES2 needs
  `OES_vertex_array_object` or a fallback path.
- **Cheaper partial fix:** skip the closing disable/unbind and track which
  attributes are currently enabled.
- **Risk:** medium-high (GLES2 / Android path needs testing).

### 2. Uniform lookup by name
- **Where:** `irr/src/OpenGL/MaterialRenderer.cpp:311` (`getPixelShaderConstantID`).
- **Problem:** linear string compare across all uniforms of the program.
  `CachedShaderSetting::set` (`src/client/shader.h:108`) returns early when the
  value is unchanged, so the lookup only runs when a value changes. Impact is
  probably smaller than I first claimed.
- **Clearest waste:** `CachedStructShaderSetting::set` (`src/client/shader.h:~202`)
  builds a new `std::string` per struct field on every update
  (heap allocation + lookup).
- **Fix:** pre-build the full field names in the constructor and cache the
  resolved IDs per shader program.
- **Risk:** low. Good first change.

### 3. All shader constants re-sent every draw
- **Where:** `MaterialRenderer.cpp:131` (`OnRender`), called from
  `setRenderStates3DMode` (`Driver.cpp:1204`).
- **Problem:** `OnSetConstants` runs on every draw. Value caching avoids the GL
  calls but still runs the callback and a comparison per uniform.
- **Fix:** separate per-draw values (world matrix) from per-frame values
  (fog, lights, time).
- **Risk:** medium.

### 4. Full material copy and compare per draw
- **Where:** `Driver.cpp:1089` (`setMaterial`), `:1181` (`LastMaterial != Material`).
- **Problem:** copies the whole `SMaterial`, loops over texture units rebuilding
  texture matrices (a 4x4 multiply for render-target textures), then compares
  member by member including all 4 texture layers.
- **Fix:** version counter or dirty flag; hash comparison.
- **Risk:** low-medium. May turn out to be noise.

### 5. Texture parameter re-checks
- **Where:** `Driver.cpp:1367` (`setTextureRenderStates`).
- **Problem:** about 7 cached-state comparisons for each of the 4 texture units
  on every material change. Mostly CPU time.
- **Fix:** GL sampler objects.
- **Risk:** medium.

### 6. Scene manager sort key
- **Where:** `irr/src/CSceneManager.cpp:509`.
- **Problem:** solid nodes are sorted by a hash of material 0 only, so nodes
  sharing the first material but differing in other slots get interleaved.
  Mostly affects entities, not the map.
- **Side note:** `CSceneManager.cpp:421`, the `ESNRP_GUI` case has no `break`
  and falls through into `ESNRP_NONE`. Harmless today, but a latent bug.

## Smaller notes
- `TEST_GL_ERROR` calls `glGetError` after every upload when error testing is
  on; this can stall the GPU pipeline. Verify it is off in release builds.
- `drawBuffers` binds and unbinds the weight buffer on every skinned-mesh draw.

## How to measure

Profiling is blocked in the current WSL environment:
- No GPU (`/dev/dxg` missing, renderer is `llvmpipe`, a CPU rasterizer), so
  profiles would be dominated by rasterization, not driver overhead.
- Build dependencies not installed: SDL2, libpng, libjpeg, zstd, LuaJIT,
  FreeType, OpenAL, Vorbis/Ogg, `ninja`, `perf`.

Plan:
1. Profile on a machine with a real GPU (native Linux or Windows).
2. Build with `-DCMAKE_BUILD_TYPE=RelWithDebInfo`.
3. Run a busy scene (many entities, long view range) and record the F6
   profiler's `draw meshes [ms]` plus a `perf record` / `perf report`.
4. Only pursue an idea if the driver draw path shows up as a real hotspot.
5. Re-measure after each change, one change at a time.

Install line for the missing packages (Debian/Ubuntu):

```
sudo apt install -y ninja-build libsdl2-dev libpng-dev libjpeg-dev \
  libzstd-dev libluajit-5.1-dev libfreetype-dev libopenal-dev \
  libvorbis-dev libogg-dev linux-tools-generic
```

## Not worth it

Switching language (Rust, Zig, Odin, Carbon, Cpp2, ...) would not speed these
up: the cost is the number of GL calls and per-draw CPU work, not the language.
For safety instead, use ASan/UBSan builds, `-D_GLIBCXX_ASSERTIONS`,
`clang-tidy`, and `std::span` / `std::string_view`.
