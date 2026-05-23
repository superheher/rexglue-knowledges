# Codegen & first build (Phase 2)

rexglue **0.8.1.4-dev** turned `private/extracted/default.xex` into C++ via
`rexglue codegen` (manifest auto-discovered). Toolchain: Clang 22.1.6,
RelWithDebInfo, Ninja, D3D12 backend.

## Scaffold (rexglue init)

`rexglue init --project-name south_park_td --xex-path private/extracted/default.xex
--game-root private/extracted`. Note rexglue **requires the XEX inside the
game-root**, so the entrypoint is the copy at the root of the extracted asset
tree (not the standalone `private/default.xex`). Generated: `CMakeLists.txt`,
`CMakePresets.json` (Ninja, per-config win/linux × debug/release/relwithdebinfo),
`south_park_td_manifest.toml` (`[project] game_root`, `[entrypoint] file_path ->
generated/default`), `src/main.cpp`, `src/south_park_td_app.h` (derives
`rex::ReXApp`).

## Codegen output

| Measure | Value |
|---|---|
| Image base / size | `0x82000000` / `0x930000` |
| Code base / size | `0x82100000` / `0x4F0C18` (~5.0 MiB) |
| Functions emitted | **20,045** |
| Translation units | **51** `south_park_td_recomp.*.cpp` (+ init/register) ≈ **102 MiB** C++ |
| Codegen time | ~7 s |

## Analysis warnings (taxonomy & counts)

rexglue converged **without `--force`** (it auto-stubs). Warning histogram from
the codegen log (counts approximate):

| Warning | Count | Meaning / disposition |
|---|---:|---|
| Unresolved conditional branch to 0x# | 15,927 | analyzer couldn't bind a branch target to a known block; mostly data-as-code & boundary noise |
| Unresolved `b` target | 2,514 | as above, unconditional |
| Unresolved function (no CallTarget) | 2,436 | emitted as `REX_FATAL` call stubs (compile OK, trap at runtime) |
| Analyze: function not in any code region | 1,465 | spurious functions outside `[0x82100000, 0x825F0C18)` |
| `Unable to decode instruction` | 1,471 | data embedded in `.text` (e.g. const pools/jump tables) read as code |
| `Function 0x# has no blocks` (stub) | 278 | out-of-image / sentinel addrs (`0xFFFFFFFF`, `0xFExxxxxx`) — import/placeholder |
| Unimplemented PPC instr | ~600 | `lfqu`/`lq`/`stfqu`/`lmw`/`stfq` (load/store multiple & FP-quad), `ba`/`bla` (absolute branch), a few `vmsumshs`/`vmhaddshs` (VMX) — become `UNIMPLEMENTED` runtime traps |
| `Cannot resolve ordinal` (imports) | 74 | 42 `xboxkrnl` + 32 `xam` import slots rexglue's runtime db doesn't auto-map; see [[50-imports-backlog]] |
| `REX_FATAL` unresolved-call stubs (generated) | 18,376 | compile fine; trap only if a real boot path reaches one |

These are *expected* for a first pass on a 2009 XDK build (R3). The vast majority
sit in spurious/data regions; the real boot path exercises a small subset, fixed
case-by-case in Phase 3 via `[functions]`/`[[switch_tables]]`/`[[invalid_instructions]]`.

## Compile blocker fixed: undeclared-label gotos

The **only** thing that stopped the build (not just warnings) was
**70 `goto loc_XXXX;` to labels never defined in their function** (57 distinct,
43 functions) → clang `use of undeclared label`. Root cause: when a caller's
PDATA/declared range **overlaps another function's entry**, rexglue's
analysis-time resolver prefers "internal label" (it trusts the declared size in
`containsAddress`) over "function entry", so it emits a local `goto` but never
emits the label (the block belongs to the other function). 19/70 target a real
function entry (should be tail calls); 51/70 target a non-entry address with no
emitted block (data-in-code / boundary gap).

Fix (reproducible, re-run after every codegen): `tools/scan_recomp.py` measures
it; `tools/fix_recomp_labels.py` rewrites each, preserving any branch condition:

- target is a defined function → `sub_X(ctx, base); return;` (correct **tail call**)
- otherwise → `REX_FATAL("…"); return;` (**runtime trap** to triage in Phase 3)

After the fixup: **0 undeclared-label gotos**. This is a workaround for a rexglue
0.8 analysis limitation; the proper upstream fix is to resolve function-entry
targets before internal-label in `FunctionGraph::tryResolveFunction` (see
[[../../general/50-cpu-recompilation]]).

## Build

`cmake --preset win-amd64-relwithdebinfo` → finds the installed ReXGlue SDK via
`find_package` (CMake user package registry). `cmake --build --preset
win-amd64-relwithdebinfo` compiles the 51 recomp TUs (-O2 -g, msvcrt) + links
`rexruntime`.

### Second link blocker: `sub_0` undefined

After the label fixup, all 51 TUs compiled and the **link** failed on a single
undefined symbol `sub_0`. rexglue emits the func-table entry `{ 0x0, sub_0 }` and
`DECLARE_REX_FUNC(sub_0)` for the **address-0 null sentinel** but never emits its
body (its own low-address sentinels `sub_00000001..3` are empty stubs that *are*
emitted). Fix: a no-op weak `sub_0` body in `src/stubs.cpp` (ours, so it survives
regeneration) added to `CMakeLists.txt`.

## Status: ✅ links

`south_park_td.exe` — **36.8 MiB**, RelWithDebInfo, built with Clang 22.1.6,
links `rexruntime`. **Phase 2 acceptance met** (compiles + links to an
executable; running is Phase 3). Reproduce: `rexglue codegen` →
`python tools/fix_recomp_labels.py` → `cmake --build --preset
win-amd64-relwithdebinfo`.
