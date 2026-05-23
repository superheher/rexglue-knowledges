# Toolchains — what exists and how to choose

Four projects matter for Xbox 360 static recompilation. This is a title-agnostic
comparison and a decision guide; for the choice made on a specific port, see that
title's case study.

## The options at a glance

| Project | Role | Gives you | Notably does *not* give you |
|---|---|---|---|
| **XenonRecomp** (hedge-dev) | PPC (Xenon) → C++ recompiler | CPU translation; XEX decrypt/decompress; XEXP patching; `XenonAnalyse` (jump tables); the clearest **config spec** (save/restore, longjmp, boundaries, invalid-instr, switch tables, mid-asm hooks) | **No runtime** ("making the game work is your responsibility"); no MMIO/XMA; no exceptions |
| **XenosRecomp** (hedge-dev) | Xenos shaders → HLSL → DXIL/SPIR-V | Shader translation; XXH3 shader-cache builder | Heavily **Unleashed-specific** ("do not expect it to work out of the box") |
| **rexglue-sdk** (ReXGlue) | Integrated SDK: codegen **+ runtime** | Phased PPC→C++ codegen; **D3D12 *and* Vulkan** renderer with **own shader translation**; **XMA**+SDL audio; SDL input; VFS; kernel/XAM; `rexglue init` scaffolder; **PSReX** PowerShell lifecycle; bundled `powerpc-none-elf` binutils | Maturity — **early development**; still "not a turnkey solution" |
| **rexdex's recompiler** | OG 360 PPC→C++ recompiler | The original proof of concept; historical/reference value | Older approach; less active |

All build with / target **Clang** (compiler-specific intrinsics & codegen); MSVC
and GCC are not the supported path for the output.

## How to choose

```
Need a runtime too (kernel/GPU/audio/input), not just code translation?
├─ Yes → start with rexglue-sdk (integrated; scaffolder; D3D12/Vulkan; XMA).
│        Keep XenonRecomp/XenosRecomp as reference + fallback.
└─ No / you already have a runtime (e.g. forking the Unleashed runtime)
         → XenonRecomp (+ XenosRecomp) into your own host.

On Windows and want the smoothest scripted lifecycle? → rexglue + PSReX.
Hit a rexglue rough edge that blocks a milestone? → fall back to
XenonRecomp + custom runtime for that piece (the proven Unleashed recipe).
Shader mistranslated by the runtime path? → cross-check with XenosRecomp's
prebuilt-cache path.
```

**Default recommendation for a fresh port:** drive with **rexglue-sdk** (it
removes the biggest cost — writing a host from scratch) and keep
**XenonRecomp + XenosRecomp** vendored as a reference and fallback. Their
documentation is the canonical spec for the per-title knobs even when you run
rexglue, and `XenonAnalyse` is a useful second opinion on jump tables.

## Build prerequisites (host)

| Tool | Needed for |
|---|---|
| **Clang 18–20+** (clang-cl on Windows / VS2022 "C++ Clang tools") | Building the recompilers **and compiling the recompiled output** |
| **CMake 3.25+** | All builds |
| **Ninja** | Build system |
| **PowerShell 7 (`pwsh`)** | rexglue's PSReX lifecycle (Windows) |
| **Python 3** | Extraction / asset tooling |
| **Git** | Submodules / VCS |
| Vulkan SDK | *Optional* — only for the Vulkan backend (D3D12 is primary on Windows) |

> The usual gap on a fresh machine is **Clang** — install LLVM/Clang 20+ first.

## rexglue per-title workflow (documented form)

```pwsh
# one-time: build/install the SDK
cmake --preset win-amd64
cmake --build --preset win-amd64 --target install

# scaffold a title project
rexglue init --app_name <app> --app_root <port-dir>
#   -> CMakeLists.txt, CMakePresets.json, src/main.cpp, src/<app>_app.h,
#      <app>_config.toml, generated/rexglue.cmake

# point config at the extracted XEX, then iterate
#   edit <app>_config.toml: file_path = "private/default.xex" (+ patch path)

# generate C++ from the XEX (phased codegen; --force to push past unresolved)
cmake --build --preset win-amd64-debug --target <app>_codegen

# compile generated sources + runtime into the app
cmake --build --preset win-amd64-debug
```

PSReX wrappers: `rex-configure` / `rex-build` / `rex-test` / `rex-format` /
`rex-lint`, plus `Invoke-ReXSetup`.

Per-title manual work is **expected, not optional**: function boundaries, missing
kernel/XAM imports, iterating on validation errors, and title-specific render
quirks (see `50`/`60`/`70`).

## Versioning caution

rexglue-sdk is **evolving fast** (early development). Pin the submodule commit,
bump **deliberately**, and **re-test** after each bump — the public API and
codegen output can change. XenonRecomp/XenosRecomp are more stable but tuned
around their flagship title; treat their game-specific code as a reference, not a
drop-in.
