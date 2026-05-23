# Toolchain — roles, requirements, decision

Three upstream projects are vendored as submodules under `third_party/`. This
note records what each does, what it needs, and **which one we drive with**.

## At a glance

| Project | Role | Gives us | Notably does *not* give us |
|---|---|---|---|
| **XenonRecomp** | PPC (Xenon) → C++ static recompiler | CPU translation; XEX decrypt/decompress; XEXP patching; XenonAnalyse (jump tables); mid-asm hook & function-override patterns | **No runtime** ("making the game work is your responsibility"); no MMIO/XMA; no exceptions |
| **XenosRecomp** | Xenos shader microcode → HLSL → DXIL/SPIR-V | Shader translation reference; shader-cache builder (XXH3) | Heavily **Unleashed-specific** ("do not expect it to work out of the box") |
| **rexglue-sdk** | Integrated SDK: codegen **+ runtime** | PPC→C++ codegen (phased), **D3D12 *and* Vulkan** renderer with **own Xenos shader translation**, **XMA**+SDL audio, SDL input, VFS, kernel/XAM objects, `rexglue init` scaffolder, **PSReX** PowerShell lifecycle, bundled `powerpc-none-elf` binutils | Maturity — it is **early development**; still "not a turnkey solution" |

## Decision

**Drive with `rexglue-sdk` as the primary toolchain.** Rationale:

- It is the only one of the three that ships a **runtime**, which XenonRecomp
  explicitly leaves to you. Building a host from scratch is the bulk of the work
  it removes.
- It has a clean, scriptable **per-title workflow** (`rexglue init` →
  `*_config.toml` → `cmake --build --target <app>_codegen` → build) and a
  **PowerShell** lifecycle (PSReX: `rex-configure` / `rex-build` / `rex-test` /
  `rex-format` / `rex-lint`) — a direct fit for this Windows 11 host.
- It targets **D3D12** (ideal for the native Windows host) *and* Vulkan, and
  performs **its own shader translation**, so XenosRecomp's per-game shader
  surgery is not on the critical path.
- It bundles the **PPC binutils** and a phased analyzer (discover → scan →
  register → merge → gap-fill → validate) with vtable/signature scanners, which
  helps with the function-boundary/jump-table problem (risk R3 in
  [[00-feasibility-analysis]]).

**Keep XenonRecomp + XenosRecomp as references / fallback / cross-check:**

- XenonRecomp's docs are the clearest spec of the per-title knobs (register
  save/restore addresses, `longjmp`/`setjmp`, `functions`, `invalid_instructions`,
  switch tables, mid-asm hooks) — invaluable when debugging rexglue codegen.
- `XenonAnalyse` is a second opinion for jump-table discovery.
- XenosRecomp's shader-cache approach is the fallback if rexglue's runtime
  shader path mishandles a specific shader.
- If rexglue's early-development rough edges block a milestone, the
  XenonRecomp + custom-runtime path (the proven *Unleashed Recompiled* recipe)
  remains available.

## Build requirements (host prerequisites)

| Tool | Needed for | Status on this host (2026-05-23) |
|---|---|---|
| **Clang** 18–20+ (clang-cl on Windows / VS2022 "C++ Clang tools") | Building the recompilers **and compiling recompiled output** (Clang-specific intrinsics/codegen) | **MISSING — install (R1)** |
| CMake 3.25+ | All builds | present (4.3.1) |
| Ninja | Build system | present (1.13.2) |
| PowerShell 7 (`pwsh`) | PSReX lifecycle | Windows PowerShell 5.1 present; install `pwsh` 7 for PSReX |
| Python 3 | Extraction/asset tooling | present (3.10.11) |
| Git | Submodules / VCS | present (2.53) |
| Vulkan SDK (optional) | Vulkan backend / glslang at build | optional (D3D12 is primary on Windows) |

> **Action:** the single hard prerequisite gap is **Clang**. Everything else is
> present or optional. Installing LLVM/Clang 20+ unblocks Phase 0.

## The per-title workflow (rexglue, as documented)

```pwsh
# 0. one-time: build/install the SDK (clang 20+, cmake 3.25+, ninja)
cmake --preset win-amd64
cmake --build --preset win-amd64 --target install

# 1. scaffold the title project
rexglue init --app_name south_park_td --app_root <south-park-recomp>

#    -> CMakeLists.txt, CMakePresets.json, src/main.cpp, src/<app>_app.h,
#       <app>_config.toml, generated/rexglue.cmake

# 2. point config at the extracted XEX, set boundaries/imports iteratively
#    edit <app>_config.toml: file_path = ".../private/default.xex"

# 3. generate C++ from the XEX (phased codegen; --force to push past unresolved)
cmake --build --preset win-amd64-debug --target south_park_td_codegen

# 4. compile the generated sources + runtime into the app
cmake --build --preset win-amd64-debug
```

Per-title manual work (expected, not optional): function boundaries, missing
kernel/XAM imports, iterating on validation errors, and game-specific render
quirks. This is the loop the `/goal` driver prompt automates.

## Pinned versions

Submodule commits are pinned by the super-repo gitlinks (see `git submodule
status`). rexglue-sdk is currently around **v0.8.0**. Because it is evolving
fast (R2), bump deliberately and re-test after each bump.
