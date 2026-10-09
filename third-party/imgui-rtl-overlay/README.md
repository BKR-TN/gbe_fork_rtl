# imgui-rtl overlay integration

This folder holds the pieces needed to build gbe_fork's experimental overlay against the
RTL Dear ImGui fork (`third-party/imgui-rtl`, branch `rtl_support`).

## Why the overlay needs patching

The overlay does not build Dear ImGui itself: it links `ingame_overlay`, the third-party
overlay library from `third-party/deps/common/ingame_overlay`, which vendors its own copy of
Dear ImGui (1.92.0) together with overlay-specific backend patches (precompiled DX shader
blobs, Win32 `GetKeyState` injection, an X11 platform backend, ...).

`premake5-deps.lua` therefore, right before building `ingame_overlay`:

1. copies the Dear ImGui **core** from the `third-party/imgui-rtl` submodule over
   `<deps>/ingame_overlay/deps/ImGui`, keeping the overlay-patched `backends/` and
   `imgui_win_shader_blobs.*` intact;
2. copies the fork's `misc/freetype` and `misc/rtl` addons into the same tree;
3. patches the vendored DX12 backend's legacy `ImGui_ImplDX12_Init()`: its header comments
   out `ImGui_ImplDX12_InitInfo::SrvDescriptorHeap` but the legacy body still assigns it, and
   that body is only compiled when obsolete functions are enabled (see step 5);
4. copies the generated libraqm headers from `raqm-gen/` next to the extracted libraqm
   sources (`<deps>/libraqm/src`);
5. appends a block to `ingame_overlay/CMakeLists.txt` that compiles
   `misc/freetype/imgui_freetype.cpp`, `misc/rtl/imgui_rtl.cpp` and libraqm's `raqm.c`
   into the overlay library and links it against the dependency builds of
   FreeType, HarfBuzz and SheenBidi (`<deps>/{freetype,harfbuzz,sheenbidi}/install<arch>`),
   and that removes `IMGUI_DISABLE_OBSOLETE_FUNCTIONS` from the target. Obsolete functions
   must be available because the fork's core still references the `ImDrawListFlags_*` aliases
   and because the 1.92-era backends use other obsolete APIs (e.g.
   `ImDrawData::CmdListsCount`, `ImDrawCallback_ResetRenderState`).

The block is re-written on every `--build-ingame_overlay` run (it is delimited by the
`GBE_RTL_IMGUI_BEGIN` / `GBE_RTL_IMGUI_END` markers), so editing the template and rebuilding
the deps is enough to update an already-extracted `ingame_overlay`.

The result: the overlay library, and everything that includes its installed
`InGameOverlay/ImGui/imgui.h`, compiles against the RTL fork, with the shaper attached
automatically (`IMGUI_ENABLE_RTL`).

## raqm-gen

`raqm-gen/` contains the two files that libraqm generates at build time and that
`raqm.c` / `raqm.h` include directly:

- `config.h` — selects SheenBidi as the bidi backend (the overlay already ships SheenBidi);
- `raqm-version.h` — generated from libraqm's `raqm-version.h.in` (0.11.0).
