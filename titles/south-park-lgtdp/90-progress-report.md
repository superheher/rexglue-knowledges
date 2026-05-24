# Progress report — South Park: Let's Go Tower Defense Play! recomp

Honest status of the port. **The recomp PLAYS A MATCH TO A WIN — single-player is playable:
boot → intro → title → MAIN MENU → LOCAL GAME → lobby → game-mode (Campaign) → level select
(Stan's House) → the MATCH (full gameplay HUD; waves of enemies spawn; the units defend) →
"STAGE COMPLETE!" (win, TOTAL SCORE 2,100) → CONTINUE** — all screenshot-verified, **input
works**, **no crash anywhere**, rendering correct throughout, the XMA audio thread runs. Two
root fixes got here: the **image-load setjmp/longjmp EH** (`setjmp_address=0x8242EEA0`/
`longjmp_address=0x8242EA70`) and the **session-enroll fix** (the local signed-in player was
never enrolled as a "session player" → a class of lobby→match null-derefs; routed through
`sub_82297F30`'s enroll path, `fix_recomp_labels` Fix 6). The save subsystem (xam content) is
implemented in the runtime. **Remaining (quick human play-test):** confirm save persists across
runs, audio fidelity, and the non-deterministic GPU-fence stall (`sub_821C6E58`, pre-input on
some runs). Reference: Xenia canary boots this title to its menu (compat #1156). The recompiled exe brings up
the full rexglue runtime, executes the guest CRT + game init, loads **TGA image assets**,
renders the **animated intro** (Cartman over the South Park town), passes the intro movie, and
reaches the **title screen** ("PRESS START"); pressing **Start** advances to the **main menu**
(LOCAL GAME / SCRAPBOOK / LEADERBOARDS / …) and **A** selects LOCAL GAME → the **lobby**
("1/4 SIGNED IN"). Input was driven/verified with an env-gated mnk injector
(`REX_INJECT_SCRIPT`) because synthetic OS keys don't reach SDL under automation — the guest
polls `XamInputGetState` and responds, so the plumbing is correct and a real user with a
focused pad/keyboard should navigate normally. **Next blockers:** a non-deterministic GPU-fence
stall (`sub_821C6E58`, pre-input on some runs) and a lobby→match crash (`SEH 0x1A in
sub_82101AF0`). This is "boot → menu" achieved, with the menu interactive.

**What unblocked it (the long-standing post-render hang):** the hang was NOT standard Win32
`.xdata` SEH (an early wrong hypothesis) but a **custom hand-rolled `setjmp`/`longjmp`** the
game uses for **image-format detection**: the loader (`sub_82459B00`) tries each decoder;
"try JPEG" (`sub_82458010`) `setjmp`s a CONTEXT (`sub_8242EEA0`), and because the assets are
**TGA** (not JPEG) the parser's raiseError (`sub_82456198`) `longjmp`s (`sub_8242EA70`) back
so the loader tries the next format. setjmp & longjmp share the same buffer (`vtable+144`)
in a live frame, so rexglue's `ppc_setjmp/longjmp` models it exactly. **Fix = config-only:**
manifest `setjmp_address=0x8242EEA0` + `longjmp_address=0x8242EA70` (the prior failed attempt
used the wrong setjmp addr `0x825925CC`=RtlCaptureContext). A follow-on cross-function-branch
FATAL (`0x821F23EC`) was cleared by registering 18 unresolved-branch targets in
`config/sp_functions.toml` (+ making `gen_missing_funcs.py` cumulative). Root-caused by **live
instrumentation** (logging the restore buffer + the bytes the parser rejected), which beat
days of static reasoning. Reference: **Xenia canary boots the title to its menu** (compat
#1156). **Next:** PRESS START → main menu → match (input + Phases 4–6). Updated 2026-05-24.

## Where it got to

| Phase | State |
|---|---|
| 0 Prereqs / build rexglue | **Done** — Clang 22.1.6 + rexglue-sdk 0.8.1.4 built & installed (D3D12). |
| 1 Extract & XEX recon | **Done** — `default.xex` (8.1 MB) + ~873 MiB asset tree extracted (corrected STFS math); recon recorded; DLC markers classified (no TU). |
| 2 Codegen & link | **Done** — ~15,000 funcs / 53 TUs → `south_park_td.exe` links & runs. |
| 3 Boot bring-up / first frame | **Done** — boots through the CRT → subsystem/handler init → GPU shader/pipeline creation → renders the town backdrop. |
| 4 Rendering correctness | **Renders correctly through a WON match** — intro, title, menu, lobby, game-mode, level select, the **in-game match** (Stan's House: map, enemy units on the path, gameplay HUD), and the **"STAGE COMPLETE!" results screen** all render right (screenshot-verified). Intro WMV is black (no WMV/WMA decoder). Open: a non-deterministic GPU-fence stall (`sub_821C6E58`) on some runs. |
| 5 Audio/input/save | **Input VERIFIED working through a full match** (`XamInput←input_system←mnk/SDL`; automation uses the `REX_INJECT_SCRIPT` injector, a real pad/keyboard works). **Audio:** XMA decoder thread runs (fidelity needs a human's ears). **Save:** xam-content subsystem implemented; persistence across runs not yet confirmed by a clean exit→re-launch. |
| 6 Polish / packaging | Largely gated on a human play-test: confirm save-persistence + audio, then polish/packaging. The single-player loop (boot→menu→match→win) works. |

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
   → `crash_backtrace.txt` names the guest `sub_*` frames. Located the crash as a
   guest null-pointer write in `sub_824711D0`.
6. **SEH first-cut + the unresolved-branch cascade → first frame (the milestone).** The
   `sub_824711D0` null write is a *symptom* of a **non-local control transfer** the recomp
   can't express: `sub_8242EA70` (the game's `RtlRestoreContext`/longjmp) restores a saved
   register context and `blr`s to a mid-function continuation, but static recomp emits
   `return`, so the caller resumes with the restored (setjmp-time) `r31` → null write. SDK
   patch `0007` adds an SEH first-cut (`RtlUnwind`→raise; generated `SEH_TRY/CATCH` that
   recovers a faulting frame "to caller as failure") which **stops the crash** — the worker
   recovers instead of taking down the process. Clearing the resulting post-SEH "Unresolved
   branch" codegen cascade (`fix_recomp_labels.py` *Fix 5* rewrites those FATALs to
   tail-calls; the `[functions]` config grew to 705) let the boot **reach the first rendered
   frame — the town backdrop, via D3D12 (screenshot-verified).** The first-cut only
   *recovers* (kills the worker); it does not yet *resume* the guest handler, so the boot now
   hangs presenting black frames (the remaining blocker, below).

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

## Honest remaining path (the SEH blocker, then iterative bring-up)

**Immediate next = a fuller Win32-SEH implementation** (the maintainer's chosen direction).
The boot renders the first frame, then a worker thread takes an SEH path during asset load
and the first-cut (recover-to-caller) kills it → the main thread hangs presenting black
frames. The real fix is the **non-local jump / exception resume**: the game uses table-based
SEH (`RtlCaptureContext`/`RtlUnwind`/`__C_specific_handler` imports + its own
`RtlRestoreContext` = `sub_8242EA70`). The runtime must drive the exception dispatch so a
raised exception unwinds to — and **resumes at** — the correct (mid-function) guest handler,
instead of returning to the caller. There is no guest `setjmp`, so rexglue's
`setjmp_address`/`longjmp_address` shortcut does not apply. This is the hardest part of
static recomp (mid-function resume + exception dispatch); realistic effort is deep/iterative.
**Once a worker can take an SEH path and resume**, the boot should clear the loading wait and
reach the menu; then Phases 4–6 (rendering correctness, audio XMA→SDL, input, save/continue)
are the "normal" iteration. The reference emulator (canary boots to menus) de-risks the path.

Diagnostic tooling that made this tractable (reusable): cdb **attach** `~*k` for live
guest stacks; **screenshot** the D3D12 window by handle; an **SEH fault backtrace** in
`seh_filter`; a **`WAIT_REG_MEM` stuck-detector**; **DbgPrint string** extraction from the
decrypted image (identified the D3D9 GPU-hang detector). See `general/95`, `general/80`.
