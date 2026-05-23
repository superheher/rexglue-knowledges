# Environment — host toolchain (Phase 0 record)

Recorded 2026-05-23 on the development host (Windows 11 Pro, x64, **non-admin**).

## Installed / present

| Tool | Version | How |
|------|---------|-----|
| Clang / LLVM | **22.1.6** (clang, clang-cl, lld-link) | `scoop install llvm` (user-scope) |
| PowerShell 7 | **7.6.2** | `scoop install pwsh` (user-scope) |
| CMake | 4.3.1 | pre-existing (`C:\Program Files\CMake`) |
| Ninja | 1.13.2 | pre-existing (WinGet) |
| Python | 3.10.11 | pre-existing |
| Git | 2.53 | pre-existing |
| 7-Zip | 26.01 | pulled in as a scoop dependency |
| scoop | 0.5.3 | bootstrapped (`get.scoop.sh`), user-scope |

VS 2022 Build Tools are present (MSVC 14.44 / 14.50, Windows SDK) and supply the
host C runtime/SDK that `clang-cl` targets. Its LLVM component shipped only
`clang-format`/`clang-tidy`, **not** the clang driver — hence the scoop install.

## Not installed (optional)

- **Vulkan SDK** — not installed. D3D12 is the primary backend on this host;
  install only if the Vulkan backend is needed.

## PATH / persistence notes

- scoop persisted to the **user PATH** (registry): `~\scoop\shims` and
  `~\scoop\apps\llvm\current\bin`. **New terminals** get `clang`/`clang-cl`/
  `pwsh` automatically.
- An **already-running** process won't see the change until restarted (Windows
  copies the environment at process start). Within such a session, prepend
  `"$env:USERPROFILE\scoop\apps\llvm\current\bin"` and `"$env:USERPROFILE\scoop\shims"`.

## Reversibility

Everything is user-scoped and reversible: `scoop uninstall llvm pwsh 7zip`, or
remove `~\scoop` and the two user-PATH entries.

## Next (rest of Phase 0 → Phase 1)

- `git submodule update --init --recursive` to pull rexglue-sdk's nested deps
  (FFmpeg, SDL3, Vulkan-Headers, glslang, spirv-tools, dxc-bin, …).
- Build/install rexglue-sdk (`cmake --preset win-amd64` → `--target install`).
- Then Phase 1: extract `default.xex` + assets (see `10-dump-analysis.md`).
