# Template: implementing Win32 SEH dispatch in a static recompilation

A reusable guide for the **hardest single subsystem** in static-recompiling an Xbox 360
(or any MSVC/Win32) title: **structured exception handling (SEH)**. Distilled from the
South Park: LGTDP port (see `titles/south-park-lgtdp/40-seh-implementation-plan.md` for the
worked example). If the title uses SEH, budget **multi-week**; this is where most ports stall.

## 1. Detect it
You have this problem if **all** of:
- The recompiled imports include `RtlCaptureContext`, `RtlUnwind`, `__C_specific_handler`
  (and often `RtlRaiseException`), and the binary has its own `RtlRestoreContext`.
- The boot renders the first frame(s) then **hangs presenting black frames** (a loader/init
  worker takes an error/recovery path and never completes; the main thread waits on it).
- A symbolized fault backtrace shows a guest-null write (`base+0`) a few frames under the
  thread-start trampoline, in a parser/loader, with a nonvolatile register (e.g. `r31`) = 0.
- The `.xdata` `__except`/`__finally` **handlers are emitted as SEPARATE functions**
  (funclets, `sub_<handler>`), not labels inside their parent.

## 2. The model (why a static recomp can't "just" do it)
MSVC uses **table-based SEH**: per-function `.pdata`/`.xdata` records describe `__try` scope
ranges, filters, and handler funclets. On a fault/`RaiseException`, the OS
`RtlDispatchException` walks `.pdata` frame-by-frame, calls each frame's language handler
(`__C_specific_handler`) with the scope table; on `EXECUTE_HANDLER` it `RtlUnwind`s to the
handler frame (running `__finally`s) and resumes in the **handler funclet**, then continues
**after the `__try`** via a saved CONTEXT (`RtlRestoreContext`). The resume is a **full
context switch to an arbitrary (mid-function) PC**, which a static recomp emits as a plain
C++ `return` → it returns to the caller carrying the restored (setjmp-time) registers →
corruption a few instructions later. That is the core impedance mismatch.

## 3. Dead-ends — do NOT spend time here (all proven not to work)
- **`setjmp_address`/`longjmp_address` config.** There is usually no guest `setjmp`; the
  context is captured by the SEH machinery (`RtlCaptureContext`), not a matched save. The
  recompiler's `ppc_setjmp`/`longjmp` is thread-local and `abort()`s on a no-match. Empirically
  regresses to a fail-fast.
- **Wrapping functions in host `__try`/`__except` and recovering in a wrapped ancestor.** By
  the time `RtlRestoreContext` runs, `RtlUnwind` has already unwound the `__try`-owner's frame
  off the stack — the handler is NOT a host-stack ancestor. Making `RtlRestoreContext` *throw*
  just propagates to the thread root and kills the worker (trades a crash for the same hang).
- **`catch → goto loc_<handler>`.** Fails to compile (`use of undeclared label`): the handler
  is a funclet (separate function), not a label in the parent body.
- **Filling the CONTEXT buffer alone.** Necessary but insufficient — the resume target
  (`CONTEXT.Pc`) stays 0 until the dispatch (`RtlUnwind`/`__C_specific_handler`) writes it.

## 4. The implementation (the pieces, in dependency order)
1. **`RtlCaptureContext`** (often a no-op stub) → fill the guest CONTEXT buffer from the
   calling thread's context (GPRs/FPRs/SP/CR/PC, **big-endian**; access via the thread's
   `thread_state()->context()` + `TranslateVirtual` + `byte_swap`). Verify safe in isolation.
2. **`RtlDispatchException`** (the OS dispatcher; trigger it from your fault/`RaiseException`
   path): walk the guest `.pdata` from the fault PC; for each frame, read its handler +
   scope table (`.xdata`, which the recompiler already parses at codegen — export it as a
   runtime table, or parse the in-memory guest `.xdata`); call `__C_specific_handler`.
3. **`__C_specific_handler`**: for the scope covering the fault PC, run the guest filter
   funclet; on `EXCEPTION_EXECUTE_HANDLER`, drive the unwind to that scope's handler.
4. **`RtlUnwind`**: run intervening `__finally` funclets; set the target CONTEXT (incl. the
   resume PC = the handler/continuation) into the buffer.
5. **`RtlRestoreContext`** (the guest's own `sub_XXXX`): restore the buffer's registers and
   **`REX_CALL_INDIRECT_FUNC(ctx.lr)` (call the handler funclet)** instead of `return`; then
   resume after the `__try` (the funclet's continuation).
6. **Filter/continuation correctness**: pick the right scope via the filter; resume at the
   instruction after the `__try`.
**1+2..5 are interdependent** — none unblocks alone; budget the whole subsystem together.

## 5. Verify (techniques)
- Attach cdb (`-p <pid> -c "~*k; qd"`) to the *hung* process for all guest stacks
  (`module!__imp__sub_XXXX`); screenshot the live D3D window by handle to confirm rendering;
  add a symbolized backtrace inside the SEH filter to find a recovered-but-dead worker.
  See `general/95`.
- After each piece: confirm no rendering regression; confirm the worker fault changes/clears.

## 6. Effort
This is the recompiler's hardest part — "expect to implement, not configure." Realistic:
**multi-week** for a robust dispatch, before the title reaches its menu. De-risk by getting
the title booting to a menu in a reference emulator (e.g. Xenia canary) first, so you know
the path is achievable.
