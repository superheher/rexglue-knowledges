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

## rexglue-sdk build (Phase 0 complete)

- Submodules pulled recursively; rexglue-sdk's nested deps (FFmpeg, SDL3,
  Vulkan-Headers, glslang, spirv-tools, dxc, …) are present.
- Configured + built + installed: `cmake --preset win-amd64` then
  `cmake --build --preset win-amd64-release --target install` (exit 0).
  Configure summary: **Graphics D3D12=ON, Vulkan=OFF**, Tracy=ON, C++23, Clang
  22.1.6, AVX2 (`-march=x86-64-v3`).
- Install prefix: `third_party/rexglue-sdk/out/install/win-amd64/`
  (`bin/rexglue.exe` 3.25 MB, `bin/rexruntime.dll` 11.6 MB, `lib/`, `include/`,
  `lib/cmake/rexglue/`, `share/rexglue/` scaffold sources).
- **`rexglue --version` → `0.8.1.4-dev.ge8ce24f`**; `rexglue --help` lists
  subcommands `init`, `codegen`, `recompile-tests`. **Phase 0 acceptance met.**

> Build-env gotcha (recorded for reuse): the agent's shell sessions were spawned
> *before* scoop edited the user PATH, so `clang`/`pwsh`/`rexglue` were invisible.
> Fix per command: `$env:Path = [Environment]::GetEnvironmentVariable('Path',
> 'Machine') + ';' + [Environment]::GetEnvironmentVariable('Path','User')`.

## Next → Phase 2

- `rexglue init` the project scaffold; configure `*_config.toml`
  (`file_path` → `private/default.xex`); build the `*_codegen` target; compile
  the app. See `10-dump-analysis.md` for base/entry/imports.
