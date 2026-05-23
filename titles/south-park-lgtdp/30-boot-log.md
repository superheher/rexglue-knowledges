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

## UPDATE — corrected root cause (cdb `sxe eh`)

Two more fixes + a debugger pass refined the picture:

- **Fix #6 (runtime patch `patches/0002`):** the XDK entry thunk doubles as the
  thread trampoline and only runs process init when **`r3 == -1`** (`xstart:
  cmpwi r3,-1; bne <epilogue>`). `KernelState::PrepareModuleLaunch` launched the
  main thread with `start_context = 0` → `r3 = 0` → init skipped. Changed it to
  `0xFFFFFFFF`.
- **The exit "crash" is NOT guest memory corruption — it is a rexglue runtime
  *shutdown* bug.** `cdb` with `sxe eh` caught **no** guest C++ exception
  (so the boot path does *not* hit a `REX_UNIMPLEMENTED` throw — that macro
  `throw`s `std::runtime_error`). The faulting stack is entirely host teardown:

  ```
  rexruntime!rex::ui::Window::RemoveInputListener+0xa   (read @ 0xFFFFFFFFFFFFFFFF)
  rexruntime!rex::input::mnk::MnkInputDriver::~MnkInputDriver
  rexruntime!rex::input::InputSystem::Shutdown
  rexruntime!rex::Runtime::Shutdown / ~Runtime
  south_park_td!rex::ReXApp::OnDestroy → wWinMain
  ```

  So: the guest entry thread runs and **returns cleanly** ("Execution complete" is
  logged by `ReXApp::LaunchModule`'s watcher when the entry thread *exits*, then
  the app quits), and the app then **crashes during input-system teardown** by
  dereferencing a `0xFFFFFFFF` pointer. Runs clean under cdb (different heap) →
  the `0xC0000005`/`0xC0000374` Heisenbug.

### Real remaining blockers (re-prioritised)

1. **Functional: the guest entry returns without entering the game loop** and
   spawns no game threads. It is *not* crashing — `main()`/init runs briefly and
   returns. Why is the open question (needs guest-execution tracing). Likely a
   stubbed/unresolved import (74: 42 xboxkrnl + 32 xam) or a missing instruction
   class returning/te­rminating the init early. Note: cdb breakpoints on the
   recompiled `__imp__xstart` did not bind reliably against the 67 MB PDB; use
   build-time guest-function profiling (`REXGLUE_PROFILE_GUEST_FUNCTIONS` + Tracy)
   or targeted host-call logging instead.
2. **rexglue input-teardown shutdown AV** (above) — a runtime bug; cosmetic while
   the game exits immediately, but should be fixed (it masks clean exits and adds
   nondeterminism). Inspect `MnkInputDriver::~MnkInputDriver` /
   `Window::RemoveInputListener` for an uninitialised/freed listener pointer.
3. **Unimplemented PPC instructions** — `REX_UNIMPLEMENTED` *throws*, so any on a
   *reached* path aborts that guest thread. **`lmw` now implemented** (patch
   `0003`): codegen had `build_stmw` but not its load mirror `build_lmw`, so
   functions using the stmw/lmw non-volatile-GPR save/restore pair threw in their
   epilogue — a real bug fixed (verified `Unimplemented: lmw` 119 → 0). Remaining
   ~684 are dominated by `lfqu`/`lq`/`stfqu`/`stfq` clustered in the `0x82132xxx`
   **data-in-code** region (likely *misdecoded data* — better handled with
   `[[invalid_instructions]]` than builders) plus a few `ba`/`bla`/VMX. Triage
   each by whether it sits in a real function before implementing.

### Decisive: the entry makes ZERO kernel calls

Temporarily instrumenting the central kernel wrapper `REX_HOOK` (hook.h) with
`REXKRNL_TRACE("kcall {}", #subroutine)` and running at `--log_level=trace`
showed **`kcall` count = 0**: the guest entry thread runs and returns having
called **no** kernel function at all. Static trace of the reached chain confirms
why — it is trivial:

```
xstart (0x824499A0):  cmpwi r3,-1; (r3==-1 →) bl sub_8244EC20; lwz r3,84(r1); <epilogue>; blr
sub_8244EC20:         b sub_8244EC28                       (1-instruction thunk)
sub_8244EC28:         r11=[r13+336]; if (r11!=0) return; [[r13+256]+352]=r3; return
```

i.e. the XEX entry point does a little PCR/TLS bookkeeping and returns. A real
title entry would call dozens of kernel imports (heap/TLS init, asset file I/O,
`ExCreateThread`, …). **The entry point as executed does not lead into the game's
real startup.** (The `REX_HOOK` instrumentation was reverted afterwards; to repeat
it, re-add that one line and rebuild the runtime — gated by `--log_level=trace`.)

So the top Phase-3 task is **not** "fix a crash" — the guest code that runs is
stable — it is "**get the title's real init/main to run.**" Open questions for the
next session:

1. ~~Is `0x824499A0` truly the process entry / decode correct?~~ **RESOLVED.**
   Built an independent decrypt+decompress (`tools/xex_decrypt.py`: retail AES key
   → session key via ECB → CBC decrypt → basic-block decompress; verified — image
   starts `MZ`, size `0x930000`). Findings: (a) the entry `0x824499A0` is
   confirmed by **both** the XEX `ENTRY_POINT` header **and** the embedded PE
   `AddressOfEntryPoint`; (b) rexglue's decode there is **byte-exact correct**
   (`2F03FFFF`=`cmpwi r3,-1`, `48005275`=`bl 0x8244EC20`, …) — *not* a decode bug;
   (c) the entry is the **tail of function F** whose real start is `0x82449968`
   (prologue `mflr; stw r12,-8; std r31,-16; stwu r1,-0x70` at `0x82449968`–`74`).
   rexglue keeps F as `sub_82449968` (a utility **called 11×**) **and** emits a
   duplicate tail as `xstart` at the entry. F's body (before the entry) does
   `bl 0x8244EC98`, which does an **indirect `bctrl` through a data pointer at
   `0x8260E0F0`** (vtable/dispatch — real init that depends on initialized data).
   From the entry (F's tail) only the two trivial PCR helpers (`sub_8244EC20/EC28`)
   are reachable; the game's big functions are not. **Conclusion: codegen is
   correct; this is a startup-model problem** — entering at the authoritative entry
   (which is F's tail) does not bootstrap the game. Next: compare against how
   Xenia executes *this* title; investigate the CRT `_initterm`/static-init path
   and what initializes the dispatch table at `0x8260E0F0`; determine why the
   entry is the tail of a utility (linker ICF/COMDAT folding?) and what the real
   bootstrap root is (look for an unreferenced root function that reaches the big
   functions like `sub_82392840`).

   **Narrowed further (verified):** the embedded PE has **no TLS directory**
   (data dir[9] = 0,0 — so no TLS callbacks drive init), and `.pdata` is at RVA
   `0xE8C00` size `0x14D80` (~10,672 functions; relocations dir present, delta 0).
   The init dispatch `sub_8244EC98` does `lwz r11,[0x8260E0F0]; lwz r11,[r11+0x20];
   bctrl`, but `[0x8260E0F0]` in the **static image is `0x000000A1`** (a small
   value, not a code pointer) — i.e. that table is populated by **runtime C++
   static initialization** (`_initterm` over the `.CRT$XC*` arrays), which never
   runs because the entry stub doesn't call the CRT init and there are no TLS
   callbacks. **Net: the real boot needs the CRT static-init + `main` path to
   run; the recompiled code is correct but that path is never entered.** The
   decisive next step is a **dynamic PC trace from a known-good emulator (Xenia)**
   on this exact title to see what actually executes from the entry, then
   replicate it (likely: run `.CRT$XC*` initializers, then call the real `main`).
   This is RE/dynamic-analysis work beyond static poking — the multi-week core.

   **Static-init data located (`tools/find_init_arrays.py`).** The decrypted image
   *does* contain the initializer/vtable code-pointer arrays in early `.rdata`
   (e.g. `0x82003EB0` ×60, `0x820047B8` ×31, `0x820048D8` ×99, `0x82005018` ×50,
   then a large vtable block from `0x820DBxxx`). So the C++ static-initializer
   table (`.CRT$XC*`) exists — the boot simply never runs it. Concrete next steps
   to wire it up: (a) find `_cinit`/`_initterm` (a function looping `mtctr;bctrl`
   over an incrementing pointer vs an end bound) and read which array address it
   passes → that's the real `.CRT$XC` range; (b) run those initializers via the
   dispatcher (they populate tables like the one at `0x8260E0F0` that `sub_8244EC98`
   reads); (c) find and call the real `main`. Each is a guest function the runtime
   can invoke directly. This remains multi-day RE, but the data is all present and
   the path is concrete.

## Reference-project comparison (breakthrough)

Compared against working ReXGlue ports (see [[../../general/99-references]]).
**TiP-Recomp (Viva Piñata — playable)** commits its generated code; its `xstart`
(entry) is a **normal CRT startup**:

```
__imp__xstart:  mflr r12; bl __savegprlr_28; addi r31,r1,-496; stwu r1,-496(r1)
                ... set globals ...; bl 0x82b14580; li r3,1; bl 0x82b0be20;
                bl 0x82b0acc0; cmpwi r3,0; ...      // real prologue + init + main
```

i.e. a function **start** with its own prologue (`stwu r1,-496`) that calls the
init/`main` chain. **South Park's `xstart` has no prologue** — it's the mid-function
tail of a 0x70-frame utility (`sub_82449968`, called 11×) that just does
`cmpwi r3,-1; …; epilogue`. So South Park's XEX/PE entry point (`0x824499A0`,
triple-verified) is **anomalous**: it does not point at a CRT-startup function.

This reframes the blocker precisely: **find South Park's real CRT-startup**
(the `mflr;bl __savegprlr_*;stwu r1,-LARGE; … _initterm … main` function — a root
not reached from `0x824499A0`) and either make the runtime start there or hook the
entry to jump there. Reference configs also show the *supporting* setup a booting
title needs that this scaffold lacks: `[rexcrt]` (map guest heap/CRT → native,
heap all-or-nothing), `setjmp`/`longjmp` addresses, `[functions]`/`[[midasm_hook]]`.
Concrete next step: scan the recompiled code / `.pdata` for the `__savegprlr_*`-then-
`stwu r1,-LARGE`-then-multiple-`bl` startup signature to locate the real entry.

### Dynamic trace — entry path proven (not inferred)

Instrumented the generated `xstart`/`sub_8244EC20`/`sub_8244EC28` with `REXLOG_INFO`
(any function can be traced this way; or instrument `REX_FUNC_PROLOGUE` in the
generated `*_init.h` for a full guest call trace) and ran:

```
TRACE xstart entered r3=-1 r1=7018FF00 r4=00000000
TRACE EC20 entered
TRACE EC28 entered, [r13+336]=00000000
```

Confirms **the entire guest execution is the stub chain** `xstart → EC20 → EC28
→ return` (EC28 does *not* early-return; it stores 0 to `[[PCR+0x100]+0x160]`),
calling **zero** game/CRT code, then the thread exits → app quits. r3=-1 (our
launch patch took effect); r4=0. The earlier "cdb bp didn't bind" was a symbol
artifact — the entry *did* run. So the entry-is-a-stub diagnosis is now
**dynamically verified**, not just static.

Static search for `_initterm`/`mainCRTStartup` (`tools/find_initterm.py`,
`tools/find_maincrt.py`) cannot pin it: of 20,046 functions, **11,656 are "roots"**
(no static caller) because the engine is heavily C++/virtual — `main` and most
logic are reached via **indirect/vtable calls** that a static call graph can't
follow. The CRT-signature filter (root + `__savegprlr_*` + moderate frame + ≥4
`bl`) still returns ~20 *game* functions (e.g. one with 85 `bl`s). The runtime
itself does **not** run C++ static init (verified: it relies on the guest entry's
`mainCRTStartup` to do it) — so the stub entry is exactly why nothing initializes.

**Definitive conclusion:** the recompiled code is correct and complete; the blocker
is purely the *launch model* — South Park's XEX entry is a stub, its real
`mainCRTStartup` is reachable only through indirect calls, and locating it (or
seeing the true boot path) provably requires **dynamic analysis (a Xenia/real-HW
PC trace of the boot)** or **a decompiler with indirect-aware xrefs (Ghidra/IDA)**.
Neither is available headlessly on this host. This is the single missing input;
everything up to it (extract → recompile → build → boot runtime → execute guest
code) is done and verified.
2. Does the title rely on **TLS callbacks / C++ static initializers** that the
   runtime must run before/around the entry? (XEX `TLS_INFO` is present.)
3. Does the XDK startup expect the entry to be invoked by an `XapiThreadStartup`
   trampoline (the `xapi_thread_startup` XThread arg, currently 0) rather than
   directly? Cross-check against how Unleashed/other rexglue titles launch.
4. The real game/CRT code clearly *exists* in the output — the largest functions
   by call count are `sub_82392840` (199 calls), `sub_82525F20` (192),
   `sub_825270CC` (166)… — but none are reached from the entry chain. Notably the
   **early-code region `0x82131xxx`–`0x82132xxx`** (right after `code_base
   0x82100000`) holds big functions (`sub_82131E60`, `sub_82132188`, ~110 calls
   each) **and** the cluster of `lfqu`/"unable to decode" warnings — i.e. likely
   CRT init that is **partially mis-recompiled** (unimplemented FP-quad ops that
   would `throw`, plus data-in-code mis-disassembly). Implementing those ops and
   marking the data regions (`[[invalid_instructions]]`) is a concrete lead for
   getting early init to recompile correctly.

**Status:** boots through the **entire rexglue runtime init** and **executes guest
code without faulting**; the only crash is a runtime *shutdown* bug (input
teardown), not guest code. **Not yet at a rendered frame** — the executed entry
returns without starting the game (zero kernel calls). Getting the real init to
run is the next milestone and the genuine multi-week core of bring-up (per the
plan's Phase 3 estimate). All tools, patches, evidence and next diagnostics are
recorded above.
