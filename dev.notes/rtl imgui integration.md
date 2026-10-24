# RTL (Arabic) Dear ImGui integration

How gbe_fork's experimental overlay renders Arabic / bidirectional text, and how to build it.

## TL;DR

The overlay links `ingame_overlay`, which vendors its own Dear ImGui. We replace that
vendored **core** with the RTL fork ([`third-party/imgui-rtl`](../third-party/imgui-rtl),
branch `rtl_support`, submodule) and compile its shaper addon
(`misc/rtl/imgui_rtl.cpp`) plus **libraqm / HarfBuzz / FreeType / SheenBidi** into the same
library. `IMGUI_ENABLE_RTL` is defined while compiling that core, so the shaper attaches to
the font atlas automatically: every string the overlay passes to ImGui is shaped and
bidi-reordered without any per-call opt-in.

Nothing in the overlay source has to pre-reverse or re-shape Arabic strings — text is plain
logical-order UTF-8.

## What each piece is

### Dear ImGui RTL fork (`third-party/imgui-rtl`)

An upstream-tracking Dear ImGui checkout (currently `1.93.1 WIP`, tracking upstream master
and merged regularly) with a shaping layer, in the same style as the built-in FreeType glyph
loader:

- `imgui.h` / `imgui_internal.h` / `imgui_draw.cpp` / `imgui_widgets.cpp`:
  `ImFontShaper` interface, `ImFontAtlas::SetFontShaper()`, glyph-**index** lookup
  (`ImFontGlyph::GlyphId`, `ImFontBaked::FindGlyphByIndex()`), shaping-aware text
  measurement/rendering/wrapping and caret movement.
- `misc/rtl/imgui_rtl.{h,cpp}`: the `ImFontShaper` implementation built on **libraqm**,
  which itself combines **HarfBuzz** (OpenType shaping: contextual Arabic joins, lam-alef
  ligatures, mark positioning), **SheenBidi** (Unicode Bidirectional Algorithm) and
  **FreeType** (font data).
- `IMGUI_ENABLE_RTL` in the imgui build attaches the shaper at atlas build time;
  `IMGUI_ENABLE_FREETYPE` makes FreeType the glyph loader (required, the shaper needs
  `FT_Face`s for the same font data).
- Only text that needs it goes through the shaper: pure ASCII and other simple scripts take
  the stock codepoint path, so LTR UI text keeps its original speed.
- Untouched: the fork's own `examples/rtl_demo` is a standalone demo/self-test build
  (CMake) and is not part of the emulator build.

### ingame_overlay's vendored Dear ImGui

`third-party/deps/common/ingame_overlay/ingame_overlay.tar.gz` ships Dear ImGui 1.92.9
plus overlay-specific patches:

- `backends/*`: precompiled DX9/10/11/12 shader blobs (no runtime `d3dcompiler`), an
  extended DX12 `RenderDrawData` signature, injected `GetKeyState` for the Win32
  backend, a GLAD2 OpenGL loader path and X11/XCB platform backends (the XCB hooks are
  loaded with `dlopen`);
- `imgui_win_shader_blobs.*` and `ImGui::BufferingBar()` / `ImGui::Spinner()` helpers.

Those backends are compiled against the fork's core (they compile unmodified), so the
integration swaps only the core files and leaves `backends/` alone.

### Previous approach (removed)

`overlay_experimental/overlay/etc/rtl_text.h` (plus Arabic strings stored in visual order
and argument-order swaps in `steam_overlay.cpp`) pre-shaped text with SheenBidi before
handing it to ImGui. That duplicated what the fork does properly (no contextual Arabic
joining, no mark positioning, no caret/wrap support) and required every call site to opt in.
It has been removed; the translation header now carries normal logical-order Arabic.

## How the build is wired

`premake5-deps.lua`:

1. extracts/builds the shaping dependencies (same pattern as the other deps):
   - `freetype` — CMake, static, zlib/bzip2/png/brotli/harfbuzz disabled;
   - `harfbuzz` — CMake (amalgamated `harfbuzz.cc`), subset/raster/vector/gpu/utils
     disabled, `HB_HAVE_FREETYPE=ON` pointed at our FreeType prefix;
   - `libraqm` — source only; `raqm.c` is compiled directly into the overlay library;
   - `sheenbidi` — already present.
   Sources live in the deps branch, `third-party/deps/common/{freetype,harfbuzz,libraqm}`.
2. `apply_rtl_imgui_overlay_patch()` runs before `ingame_overlay` is configured:
   - copies the fork's core + `misc/rtl` + `misc/freetype` over
     `<deps>/ingame_overlay/deps/ImGui`, keeping the patched `backends/`;
   - copies `third-party/imgui-rtl-overlay/raqm-gen/{config.h,raqm-version.h}` next to the
     extracted libraqm sources;
   - patches the vendored DX12 backend's legacy `ImGui_ImplDX12_Init()` (its header comments
     out `ImGui_ImplDX12_InitInfo::SrvDescriptorHeap`, but the legacy body still assigns it);
   - (re)writes a `GBE_RTL_IMGUI_BEGIN` block in `ingame_overlay/CMakeLists.txt` that adds
     `imgui_rtl.cpp`, `imgui_freetype.cpp` and `raqm.c` to the library, defines
     `IMGUI_ENABLE_RTL`/`IMGUI_ENABLE_FREETYPE`/`HAVE_CONFIG_H` and links the
     FreeType/HarfBuzz/SheenBidi installs of the matching arch (32/64/arm);
   - strips the target's `IMGUI_DISABLE_OBSOLETE_FUNCTIONS` definition. Obsolete functions
     have to be available for both sides: the fork's core still refers to the
     `ImDrawListFlags_*` aliases, and the 1.92-era backends use removed APIs such as
     `ImDrawData::CmdListsCount` and `ImDrawCallback_ResetRenderState`.

`premake5.lua` adds `freetype`/`harfbuzz` to the dependency link list (they are static,
so the final targets need them) and their `install<arch>/lib` folders to the library
search paths. Since the new overlay uses XCB directly (its ImGui XCB backend and the X11
hook), `xcb` and `xcb-keysyms` are added to `overlay_link_linux`, which the two
experimental (overlay-enabled) targets link; without them the resulting `steamclient.so`
keeps undefined XCB symbols and only works if the game happens to load libxcb. The regular
targets don't need them and are left untouched.

## Building

### Linux (native)

```sh
# deps (all of them; add --clean for a full rebuild)
export CMAKE_GENERATOR="Unix Makefiles"
./third-party/common/linux/premake/premake5 --file=premake5-deps.lua \
    --64-build --all-ext --all-build --j=$(nproc) --os=linux gmake

# project files (generates proto too) and build
./third-party/common/linux/premake/premake5 --genproto --os=linux gmake
cd build/project/gmake/linux
make -j$(nproc) config=release_x64
```

### Windows (cross compiled from Linux, msvc-wine)

See [cross compiling.md](./cross%20compiling.md) for the toolchain setup. The x86 and x64
dependency builds each need their own toolchain/PATH, and `--all-ext --all-build` picks up
the new deps automatically:

```sh
export WINDOWS_SDK_PATH=/opt/msvc

export PATH=$WINDOWS_SDK_PATH/bin/x86:$PATH
./third-party/common/linux/premake/premake5 --file=premake5-deps.lua \
    --32-build --all-ext --all-build --custom-cmake=cmake \
    --cmake-toolchain=$WINDOWS_SDK_PATH/cmake/toolchain-x86.cmake \
    --custom-extractor=7z --j=$(nproc) --os=windows vs2026

export PATH=$WINDOWS_SDK_PATH/bin/x64:$PATH
./third-party/common/linux/premake/premake5 --file=premake5-deps.lua \
    --64-build --all-ext --all-build --custom-cmake=cmake \
    --cmake-toolchain=$WINDOWS_SDK_PATH/cmake/toolchain-x64.cmake \
    --custom-extractor=7z --j=$(nproc) --os=windows vs2026

./third-party/common/linux/premake/premake5 --file=premake5.lua --genproto --os=windows vs2026

cd build/project/vs2026/win
/opt/msvc/bin/x86/msbuild /nologo /v:n '/p:Configuration=release,Platform=Win32' gbe.slnx
/opt/msvc/bin/x64/msbuild /nologo /v:n '/p:Configuration=release,Platform=x64' gbe.slnx
```

### Cross-compiling quirks (msvc-wine)

CMake's MSVC show-includes probe is unreliable under wine, so `cmake_build` patches the
generated makefiles (`--filter-prefix=""` → `--filter-prefix="Note: including file:"`)
between configure and build. Without that, the dependency files contain raw
`Note: including file:` lines, which makes the *next* build fail with
`multiple target patterns` in `compiler_depend.make`, and the dependency wrapper can
mangle quoted compile definitions.

`IMGUI_USER_CONFIG` is handled the same way: the overlay patch rewrites
`IMGUI_USER_CONFIG_VALUE` so the compile definition is only the file name
(`"imconfig.imcfg"`), resolved by `imgui.h` relative to its own directory where the deps
script keeps a copy. `cl.exe` under wine cannot open unix-style absolute paths from a
quoted macro, while CMake still needs the absolute path for the source list and install
rules.

Existing dependency build directories created before these fixes must be deleted once
(e.g. `build/deps/win/vs2026/ingame_overlay/build64`), because the generated compiler files
have the empty prefix baked in.

## Fonts

The overlay builds its atlas from the glyphs used by all translation strings plus the default
ranges, then loads a user font (`overlay_appearance.font_override`) and/or the embedded GNU
Unifont fallback. Unifont contains the Arabic block and the Arabic presentation forms, so
HarfBuzz's Arabic fallback shaping produces correct joined text with it; a proper Arabic font
(e.g. the fork's `examples/rtl_demo/fonts/NotoNaskhArabic.ttf`, set as the overlay font
override) additionally gets real `GSUB`/`GPOS` shaping and mark positioning.

Shaped glyphs are requested by **glyph index**, so they are loaded on demand even when the
atlas ranges only cover the base codepoints.

### Layout: right-aligned labels and SameLine()

The shaper right-aligns an RTL string that starts a line (it draws the text flush with the work
area's right edge) while the layout cursor advances left-to-right by the text width. Since
`imgui-rtl` `77f4ccdfe` the `SameLine()` continuation of such a label goes to its **left**:
`Text(rtl); SameLine(); <widget>;` starts the widget at the line's left edge and, for
width-honouring widgets (InputText, Slider, Combo, ...), clamps its width to the space left of
the label (`anchor - spacing - line_start`), so nothing is drawn over the label. Width-ignoring
items (Button, Text, Dummy, ...) simply start at the left edge, which is already clear of the
label. `SameLine(explicit_offset)`, LTR labels and wrapped text are unchanged.

`SetNextItemWidth(w)` makes the widget hug the label (`anchor - spacing - w`), but it must be
called **before** `SameLine()`: `SameLine()` consumes the width to position the item, so
setting it afterwards only changes the size, not the position.

## Maintaining this integration

- **Updating the RTL fork**: bump the `third-party/imgui-rtl` submodule. Re-check that
  `ingame_overlay`'s patched `backends/` still compile against the new core (they use only
  public backend-facing API) and that `IMGUI_DISABLE_OBSOLETE_FUNCTIONS` is still stripped.
- **Updating ingame_overlay**: re-extract it and run the build; the patch is applied to the
  extracted copy on every `--build-ingame_overlay` (idempotent, detected via the
  `GBE_RTL_IMGUI_BEGIN` marker). If the upstream `CMakeLists.txt` changes target names or
  the imgui sources, the appended block has to be adjusted.
- **Updating libraqm/HarfBuzz/FreeType**: replace the tarball in
  `third-party/deps/common`, update its `SOURCE.txt` and, for libraqm,
  `third-party/imgui-rtl-overlay/raqm-gen/raqm-version.h`.
- **portaudio on Linux** is built with `PA_ALSA_DYNAMIC=ON` and JACK/PulseAudio/sndio
  disabled, so the static library only `dlopen`s `libasound.so.2` at runtime instead of
  adding link-time dependencies to every target.
