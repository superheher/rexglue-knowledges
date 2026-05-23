# Progress report — South Park: Let's Go Tower Defense Play! recomp

Honest status of the port. **Not yet playable.** The recompiled exe now boots through
the guest CRT and game subsystem init and reaches **GPU rendering init** (shader
translation + graphics-pipeline creation) before a guest null-pointer crash ~15 s in.
Reference: **Xenia canary boots the title to its menu** from the same base `default.xex`
(compat #1156), so the target is achievable and the remaining work is tractable iterative
bring-up. A large, reusable KB accompanies the journey. Updated 2026-05-23.

## Where it got to

| Phase | State |
|---|---|
| 0 Prereqs / build rexglue | **Done** — Clang 22.1.6 + rexglue-sdk 0.8.1.4 built & installed (D3D12). |
| 1 Extract & XEX recon | **Done** — `default.xex` (8.1 MB) + ~873 MiB asset tree extracted (corrected STFS math); recon recorded; DLC markers classified (no TU). |
| 2 Codegen & link | **Done** — ~15,000 funcs / 53 TUs → `south_park_td.exe` links & runs. |
| 3 Boot bring-up | **In progress (iterating, real progress)** — boots through the CRT → subsystem/handler init → party/session writer init → **GPU shader translation + pipeline creation** (`Translated 4 shaders`, `Created 2 graphics pipelines`, `SetInterruptCallback`), ~15 s, then a guest null-pointer write (`sub_824711D0`). Each fault is concrete and fixable; see [[35-entry-forensics]]. |
| 4–6 | Not started (gated on reaching a frame). |

## What is verified working (run, observed, logged)

The exe boots the **entire rexglue runtime** (D3D12 on RTX 3060, XMA audio threads, SDL
input, VFS mount of the assets at `game:\`, kernel module load with xboxkrnl/xam imports
resolved) **and then runs the guest**: XapiThreadStartup trampoline → CRT → the game's
subsystem/handler initialisation → a party/session "writer" subsystem → the GPU path,
where it **translates shaders and creates graphics pipelines**. Progression is verified by
the per-run logs and a symbolized crash backtrace (below), advancing from a fault at 0.9 s
(first session blocker) to ~15 s this session.

## What was fixed to get here (reusable — see [[20-codegen]], [[30-boot-log]], [[35-entry-forensics]])

Boot bring-up, highest-impact first:

1. **Corrupt extracted content (the big one).** The recomp ran on a `default.xex` my own
   `tools/stfs_extract.py` had mangled — an STFS block→offset bug dropped the per-level
   hash-table block once `block ≥ 0xAA`, so `.text` survived (entry disassembled) but
   `.data`/assets were garbage. Proven by diffing `.data` against canary. Fixed the
   iterative `block += ((b+base)/base)*f` math; re-extracted; `.data` now byte-matches
   canary. *This invalidated all prior "doesn't boot" analysis.*
2. **Boot continuation.** The XapiThreadStartup trampoline ends in `blr`; rexglue compiles
   `blr` to C++ `return`, so without a re-enter the boot stopped at the entry. Patches
   `0005`/`0006`: zero the guest stack (not `0xBE` poison) + a gated re-enter loop in
   `XThread::Execute`.
3. **Null EH hook → skipped init (the writer crash).** A setjmp-style dispatcher
   (`sub_8242EEA0`) calls a runtime hook `[0x82902438]` if set, else falls through leaving
   `r3` = a stale non-zero pointer; the hook is image-zero and never installed, so a gate
   read it as "exception taken" and **skipped the writer-init callback**, leaving a buffer
   null → null write. Fix: dispatcher returns 0 on the null-hook path ("no EH infra → run
   the body"). Persisted as `fix_recomp_labels.py` *Fix 3*.
4. **Analyzer-missed function class → `[functions]` config (the big systemic win).**
   "Call to invalid or unregistered function at 0x…" FATALs are indirect-call targets
   rexglue's analyzer never emits — vtable methods its scanner skips + computed-jump/
   adjustor-thunk targets mid-function. Registered them via rexglue's `[functions]` config
   layered through the manifest's `[entrypoint].includes` (`tools/gen_missing_funcs.py`
   generates the list; **excludes the import-thunk band** or you get `undefined symbol:
   sub_8259xxxx`). A **670-function batch cleared the entire class** and the boot leapt
   into GPU rendering init.
5. **Crash handler.** The runtime had none, so faults died silently and cdb hangs on the
   D3D12 window. Added `SetUnhandledExceptionFilter` + dbghelp `StackWalk64` in `main.cpp`
   → `crash_backtrace.txt` names the guest `sub_*` frames. Located the current crash as a
   guest null-pointer write in `sub_824711D0`.

Earlier Phase-2 plumbing (all reusable, in `tools/fix_recomp_labels.py` + runtime patches):
undeclared-label gotos (rexglue over-segmentation) → tail-call/trap rewrite; `sub_0`
null-sentinel link error → weak stub in `src/`; zero-terminated func-table starting with
the `{0x0,sub_0}` entry and out-of-range entries → range filter; `r1`-on-guard-page entry
fault → reserve headroom (`0001`); `lmw` epilogue `REX_UNIMPLEMENTED` → `build_lmw`
(`0003`); 7 `XUsbcam` import stubs. *(A `r3=0xFFFFFFFF` entry theory, patch `0002`, was
disproved against Xenia and reverted.)*

## Real effort / approach

Multiple long sessions. The decisive method was **diff against the canary oracle** (it
boots the title) and, when the divergence was inside the recomp, **instrument the recomp
itself** — a one-line `REXLOG_WARN` of the suspect pointer/gate at the faulting `sub_*`
turns a silent null-deref into a root cause in one run. Static analysis (generated C++ +
`.pdata` + image scans in Python) localised; the build/run loop verified every fix. The
recompiler (rexglue 0.8, early-development) did the heavy lifting and needed a handful of
small, reproducible accommodations — none research-grade.

## Top reusable lessons (promoted to general/)

- **Verify the inputs before blaming the recomp.** The single biggest time sink was
  debugging a recomp running on *corrupt extracted content*; a `.text`-only-correct dump
  boots far enough to look real. Cross-check `.data` against the reference emulator early.
  `general/95`, `general/25`.
- **Use Xenia/canary as the ground-truth oracle** (stock *and* canary; check the compat
  tracker first). If the reference dies at the same spot, the recomp is correct; if it
  boots and you don't, the bug is yours. `general/45`, `general/95`.
- **The analyzer misses indirect-call targets** (vtable methods + computed-jump/adjustor
  thunks) → runtime "unregistered function" FATALs → fix via the recompiler's explicit-
  `functions` config + regen, **never** registering an import-thunk address. `general/95`, `general/50`.
- **Instrument the recomp** (`REXLOG_WARN`) and **install a fault handler** (dbghelp
  backtrace) — the runtime ships neither; both turn silent crashes into located bugs.
  `general/95`, `general/80`.
- **A null runtime hook can silently skip setjmp-style try bodies**; default such a
  dispatcher to the "no-jump" value. `general/80`.
- **`.pdata` is the authoritative function table** (big-endian; machine `0x01F2` needs
  capstone BE-PPC `skipdata`). Reproducible post-codegen fixups beat hand-edits; keep
  upstream patches as files. `general/45`, `general/50`.

## Honest remaining path (tractable iterative bring-up — multi-week+)

The boot is an iterative bring-up, not a research problem: run → read the FATAL/backtrace
→ fix → rebuild → it advances. **Immediate next:** the located guest null-pointer write in
`sub_824711D0` (REXLOG_WARN the null field there + its callers). After the boot stops
faulting it reaches the render/update loop; then Phases 4–6 (rendering correctness, audio
XMA→SDL, input, save/continue) are the "normal" iteration. Realistic total to *playable*:
multi-week to a few months, dominated by Phase 4–6 correctness rather than any single
blocker. The reference emulator booting to menus de-risks the whole path.
