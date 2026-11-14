# CLAUDE.md — Luanti engine

Luanti (formerly Minetest): a C++17 voxel engine with an embedded Lua modding API, built on IrrlichtMt (`irr/`, SDL2 + OpenGL/GLES). See the parent `../CLAUDE.md` for cross-project rules.

**This checkout is the user's fork.** `origin` is github.com/animagr/luanti (branch `master`); `upstream` is github.com/luanti-org/luanti. The fork carries a small set of patches for the game in `../luanti-builders`, starting with the third-person camera. `doc/fork_changes.md` lists every difference from upstream; add to it with each patch. Keep patches small and compatible with stock clients and servers, so the game still runs on an unpatched engine. These patches stay in the fork and are not meant for upstream.

Start with `.codemapy/summary.md` for structure and hubs (`src/log.h`, `src/settings.h`, `src/util/string.h`, `src/nodedef.h` are the most-included headers).

## AI policy — read before writing code meant for upstream

`doc/developing/ai_policy.md` governs contributions to this repo:

- Generating substantial code with AI is strongly discouraged; autonomous "vibe-coding" PRs are rejected. Small tasks (completion, find-and-replace, small bugfixes, boilerplate) are fine.
- No AI-written text for humans: no generated documentation, PR descriptions, or issue comments. Grammar help on human-written text is fine.
- No AI-generated art, music, sounds, or other media.
- Exploring, explaining, reviewing, and debugging with AI are explicitly allowed. Significant AI use must be disclosed in the PR.

So when the user is preparing an upstream contribution: prefer explaining, reviewing, and locating code over writing large changes; never draft PR text or docs for them to submit as their own. For local experiments that won't go upstream, this doesn't restrict you.

## Build

Out-of-tree CMake build. C++17; minimum CMake 3.12 (`CMakePresets.json` has Debug / Release / RelWithDebInfo / MinSizeRel).

```powershell
# Windows (MSVC + vcpkg) — see doc/compiling/windows_msvc.md
vcpkg install zlib zstd curl[ssl] openal-soft libvorbis libogg libjpeg-turbo sqlite3 freetype luajit gmp jsoncpp gettext[tools] sdl2 --triplet x64-windows
cmake -S . -B build -DCMAKE_TOOLCHAIN_FILE=<vcpkg>/scripts/buildsystems/vcpkg.cmake -DCMAKE_BUILD_TYPE=Release -DENABLE_CURSES=OFF
cmake --build build --config Release
```

```bash
# Linux/macOS — see doc/compiling/linux.md, macos.md
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release -DRUN_IN_PLACE=TRUE
cmake --build build -j$(nproc)
```

Key cache options (defaults in `CMakeLists.txt`): `BUILD_CLIENT=TRUE`, `BUILD_SERVER=FALSE`, `BUILD_UNITTESTS=TRUE`, `BUILD_BENCHMARKS=FALSE`, `BUILD_WITH_TRACY=FALSE`, `ENABLE_LUAJIT`, `RUN_IN_PLACE` (TRUE by default on Windows — data, worlds, and games live beside the binary in `bin/`, `worlds/`, `games/`). Full list: `doc/compiling/README.md`.

## Test and lint

```bash
./bin/luanti --run-unittests                    # all C++ unit tests (src/unittest/)
./bin/luanti --test-module TestMapNode          # one module; name = the class's getName()
./bin/luanti --run-benchmarks                   # needs BUILD_BENCHMARKS=TRUE (src/benchmark/)

busted builtin                                  # Lua unit tests: builtin/common/tests/*_spec.lua
luacheck builtin                                # Lua lint, root .luacheckrc
luacheck --config=games/devtest/.luacheckrc games/devtest

util/test_singleplayer.sh                       # integration (Linux; client needs xvfb-run)
util/test_multiplayer.sh
```

C++ lint is clang-tidy (`.clang-tidy`, runner `util/ci/run-clang-tidy.py`). CI also enforces whitespace and optimized PNGs (`.github/workflows/whitespace_checks.yml`, `png_file_checks.yml`).

`games/devtest/` is the engine's test game — the place to add mods that exercise new or changed Lua API (its `mods/unittests/` runs Lua-side tests in-game).

## Layout

- `src/` — engine C++
  - `client/` rendering, mesh generation, camera, shaders, shadows, sound, HUD, input
  - `server/` active objects, player SAO, bans, server list; `server.cpp`, `serverenvironment.cpp`, `servermap.cpp` at top level
  - `script/` Lua↔C++ bridge: `lua_api/` (`l_*.cpp`, functions exposed to Lua), `cpp_api/` (`s_*.cpp`, engine callbacks into Lua), `common/` (conversions, `c_content.cpp`)
  - `mapgen/` v5/v6/v7/flat/valleys/carpathian/fractal/singlenode, caves, dungeons, biomes, ores, decorations
  - `network/` protocol, packet handlers, connections (`doc/protocol.txt`)
  - `gui/` formspecs (`guiFormSpecMenu.cpp`), console, touch controls
  - `database/` map/player/auth/mod-storage backends; `util/` shared helpers
  - `unittest/` C++ tests; `benchmark/` benchmarks
- `irr/` — IrrlichtMt, the forked renderer (own `README.md`)
- `builtin/` — Lua shipped with the engine: `game/`, `common/`, `mainmenu/`, `settingtypes.txt`, `async/`, `sscsm_*`
- `lib/` — vendored C/C++ libs (Lua 5.1, catch2, jsoncpp, mini-gmp, …) — don't edit
- `doc/` — `lua_api.md` (the modding API reference, ~10k lines), `client_lua_api.md`, `world_format.md`, `protocol.txt`, `direction.md` (roadmap), `breakages.md`
- `android/` — Gradle project; `util/` — CI and helper scripts; `po/` — translations

## Conventions

- C++ style: https://docs.luanti.org/for-engine-devs/code-style-guidelines/ — this codebase's style overrides `C:\Claude\CppRules.md` on naming and formatting.
- Lua style: https://docs.luanti.org/for-engine-devs/lua-code-style-guidelines/ plus `../luarules.md`.
- Tabs for indent, spaces for alignment, LF endings (`.editorconfig`).
- Any Lua API change must be documented in `doc/lua_api.md` (and `doc/breakages.md` if it breaks compatibility). Network or world-format changes must keep compatibility (protocol version, serialization version).
- Do **not** hand-edit `minetest.conf.example`, `src/settings_translation_file.cpp`, or `po/luanti.po{,t}` — maintainers regenerate them before releases. New settings go in `builtin/settingtypes.txt` only.
- Features outside `doc/direction.md` need a core-dev concept approval before a PR; open an issue first.

## Commits

Present tense, capital first letter, no trailing period, first line under ~70 chars, blank second line, then one line per point. One change per branch; never commit to `master` for a PR.
