# Boot bring-up log (Phase 3)

Chronological journal of bringing the recompiled `south_park_td.exe` from "links"
to executing guest code. Each entry: symptom → root cause → fix. This is the most
reusable artifact of the port (see [[../../general/95-pitfalls]]).

## How to run

```powershell
# rexruntime.dll + TracyClient.dll are copied next to the exe by the build.
$dir  = "south-park-recomp/out/build/win-amd64-relwithdebinfo"
$game = "south-park-recomp/private/extracted"
& "$dir/south_park_td.exe" --game_data_root=$game            # optional: --log_level=trace
# logs -> $dir/logs/south_park_td_NNN.log  (rotating, sequential)
```

`game_data_root` is a runtime cvar; the runtime mounts it as
`\Device\Harddisk0\Partition1` (guest `game:\`). Debugging:
`cdb -g -G -cf tools/cdb_cmds.txt "$dir/south_park_td.exe" --game_data_root=$game`.

## What works (runtime init — first run, clean)

The rexglue runtime initializes **fully** before guest code runs:

- D3D12 device on **NVIDIA RTX 3060** (resource binding tier 3, ROVs, tiled tier 4).
- FunctionDispatcher, SDL 3.5.0, mouse/keyboard + SDL controller input.
- Audio (XMA Decoder + Audio Worker host threads), GPU Commands + GPU VSync threads.
- VFS mounts `private/extracted` at `\Device\Harddisk0\Partition1`.
- Function table for the module: `code=82100000-825F0C18, image=82000000-82930000`.
- XEX image loads (`game:\default.xex`), variable imports patched (e.g.
  `KeTimeStampBundle`, `XexExecutableModuleHandle`, `XboxHardwareInfo`), **221 of
  325** xboxkrnl imports resolved with names; shader storage init for title
  `58410931`.

So GPU/audio/input/VFS/kernel are not blockers — the work is in the recompiled
guest code and a few runtime/codegen gaps.

## Fixes to reach guest execution (in order)

| # | Symptom | Root cause | Fix |
|---|---------|-----------|-----|
| 1 | compile: `use of undeclared label loc_XXXX` (70×) | rexglue over-segmentation / PDATA-range overlap emits `goto` to another function's entry or a missing block | `tools/fix_recomp_labels.py`: rewrite to tail-call or trap (see [[20-codegen]]) |
| 2 | link: `undefined symbol: sub_0` | address-0 sentinel declared+listed but no body emitted | weak no-op `sub_0` in `src/stubs.cpp` |
| 3 | run: `No function registered at 824499A0` (entry) — nothing registered | `func_mappings[]` starts with `{ 0x0, sub_0 }`; runtime loops until `guest==0`, so it stops on entry 0 → registers **zero** functions | drop address-0 entry (range filter in `fix_recomp_labels.py`) |
| 4 | run: `41 func_mappings entries rejected` → `Runtime setup failed: C0000001` | rexglue lists ~41 out-of-image stub funcs (`0x8260C92C..0xFFFFFFFF`); runtime aborts if any entry is outside `[code_base, code_base+code_size+thunk_reserve)` | same range filter drops them |
| 5 | run: AV in `xstart+0x94`, `mov eax,[rdx+rcx]` reading guest `0x70190068` | initial guest `r1 = stack_base`, which is the stack's `PAGE_NOACCESS` guard page; `xstart` has no prologue and its epilogue reads `[r1+104]` (caller/loader frame) **above** r1 | rexglue runtime patch: start `r1` a 16-byte-aligned `0x100` below `stack_base` (`patches/0001-...`, `thread_state.cpp`) |

cdb confirmed #5: `!address` showed `r1`/`r1+104` on a `MEM_COMMIT / PAGE_NOACCESS`
page, with `PAGE_READWRITE` just below — i.e. r1 sat exactly on the guard.

## Current blocker: entry returns early + memory corruption (Heisenbug)

After fix #5, **`xstart` executes to completion** ("Execution complete") — no more
faults inside it. But:

- The process then exits with a **nondeterministic** code: `0xC0000005`
  (access violation) or `0xC0000374` (`STATUS_HEAP_CORRUPTION`) across runs.
- **Under cdb it runs clean** to "Execution complete" and exits 0 (different heap
  layout hides the fault) → classic **heap-corruption Heisenbug**: the recompiled
  code scribbles memory during its brief run; the fault surfaces later (often on
  teardown) only in the non-debug heap.
- The entry **returns after ~80 ms** and **no guest threads are created** (no
  `XThread::Execute thid 6+`). A game's main thread should enter a render/update
  loop and not return. So recompiled `main()`/CRT path is taking a wrong branch
  early and returning, rather than running the game.

Heap (host) corruption — vs guest-memory OOB — implies a guest write is landing on
a runtime structure that stores **host** pointers inside guest memory (e.g. the
`X_KPCR` carries `host_stash = host PPCContext*`). A mistranslated/over-wide write
near such a structure would corrupt a host pointer → later AV/heap abort.

### Leading hypotheses (next steps)

1. **Mistranslated instruction corrupting memory.** ~600 ops were `UNIMPLEMENTED`
   at codegen (`lmw`/`lq`/`stq`/`lfqu`/`stfqu`/`ba`/`bla`, a few VMX). If a real
   boot-path function uses one, it traps or (worse) a *subtly wrong* translation
   writes OOB. Action: build with guest-function profiling / add targeted logging
   to find the last guest function before the corruption; check whether any
   `UNIMPLEMENTED`/`REX_FATAL` stub is hit on the boot path.
2. **Bad function boundary / jump table** on the boot path → wrong control flow →
   early return. Action: `XenonAnalyse` cross-check of switch tables; add
   `[[switch_tables]]`/`[functions]` for boot-path functions; re-rate the ~16k
   "unresolved conditional branch" warnings for ones inside reached functions.
3. **Stubbed import returns wrong value** (74 unresolved: 42 xboxkrnl + 32 xam) →
   CRT/init bails. Action: enumerate which of the 74 are *called* during boot and
   implement them (see [[50-imports-backlog]]).

### Tools added this phase

- `tools/fix_recomp_labels.py` — post-codegen fixups (label gotos + func_mappings
  range filter). Re-run after every `rexglue codegen`.
- `tools/cdb_cmds.txt` — cdb command script for crash triage.
- `patches/0001-rexglue-thread-r1-stack-headroom.patch` — runtime r1 fix.

**Status:** boots through full runtime init and executes the guest entry point;
**not yet at a rendered frame**. Next: root-cause the early-return/corruption per
the hypotheses above.
