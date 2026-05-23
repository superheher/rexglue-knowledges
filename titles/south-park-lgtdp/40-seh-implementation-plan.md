# SEH implementation plan — the boot's last blocker (resume after exception)

This is the concrete plan to clear the **current** blocker: the recomp renders the first
frame, then a worker thread faults during asset load, the SEH first-cut *recovers* it
(killing the worker), and the main thread hangs waiting on it. The real fix is making the
guest's exception path **resume** correctly. Companion to [[35-entry-forensics]];
general lessons in `general/95`, `general/80`.

## ⚠️ CRITICAL UPDATE (after deeper analysis — supersedes "Recommended approach" below)
Two findings change the plan:
1. **`RtlCaptureContext` is a no-op STUB** in the runtime (`xboxkrnl_rtl.cpp:558`,
   `"[STUB] … not implemented"`) — it never fills the buffer, so `sub_8242EA70` restores
   **garbage** (the `r31=0`). Filling it (the "setjmp" save: f14–f31@0, SP@144, r13–r31@152,
   CR@304, PC@308=`lr`, VMX@320, big-endian) is part of any fix. Get the current thread's
   `PPCContext` via `thread->thread_state()->context()`; write guest mem via
   `REX_KERNEL_MEMORY()->TranslateVirtual`.
2. **The `setjmp_address`/`longjmp_address` shortcut (approach B below) is RULED OUT.**
   rexglue's mechanism (context.cpp:170–188) special-cases the *call sites*: a `setJmpAddress`
   call saves a PPCContext snapshot + host `setjmp`; a `longJmpAddress` call host-`longjmp`s
   back **to that setjmp site**. But Win32 SEH does **not** resume at the capture site: on
   `EXECUTE_HANDLER`, `__C_specific_handler` → `RtlUnwind` → `RtlRestoreContext`
   (`sub_8242EA70`) jumps to **`buf[308]` = the `__except` block** (a *mid-function* PC),
   which ≠ the `RtlCaptureContext` return point. Host setjmp/longjmp can only resume at the
   setjmp site, so it cannot reach the handler. (It would only work if capture==resume, i.e.
   `EXCEPTION_CONTINUE_EXECUTION`, which is not the try/except case the worker hits.)
   **EMPIRICALLY CONFIRMED FAILED (2026-05-24):** set `setjmp_address=0x825925CC` +
   `longjmp_address=0x8242EA70`, regen+rebuilt → **regressed** to `0xC0000409` (fail-fast)
   + a NEW `sub_82266AC8` near-null crash. rexglue's `ppc_setjmp`/`ppc_longjmp` use a
   **`thread_local`** jmp_buf map and **`std::abort()` on no-match**; the title's
   RtlCaptureContext (2 direct sites → `ppc_setjmp` in `sub_8243FCF0`/`sub_82446CE0`, which
   set a **stack-buffer** jmp_buf then call `RtlUnwind` and continue) is **SEH machinery,
   not a same-thread/same-frame setjmp/longjmp pair** → the worker's `ppc_longjmp` finds no
   matching live jmp_buf → abort/corruption. Reverted. **Do NOT retry this shortcut.**

**⇒ Required approach = host-SEH `CATCH` runs the guest `__except` (NO external mid-function
entry).** Re-tracing the chain: the `__try`/`__except` lives in **`sub_82456198`** (the
caller of `sub_8242EA70`); its `__except` = `buf[308]` is *inside `sub_82456198`*. So the
resume can happen **within `sub_82456198`'s own generated C++ function** — its host
`SEH_CATCH` block just `goto`s the `__except` label. rexglue **already wraps** such functions
with `SEH_TRY`/`SEH_CATCH` (`generate_exception_handlers=true`, `SehExceptionInfo` scopes
from `.xdata`); the first-cut only **stubbed the CATCH** to "recover-to-caller (r3=0)". The
fix is therefore **bounded codegen + runtime**, not an architecture rewrite:
1. **Codegen (`function_graph.cpp` SEH_CATCH):** instead of recover-to-caller, dispatch the
   scope(s): run the `__except` **filter** (scope `filter` addr) with the exception info; on
   `EXCEPTION_EXECUTE_HANDLER` (1), restore the entry frame and **`goto` the scope `handler`**
   (the `loc_XXXXXXXX` for `buf[308]`, which is a label in this same function); on
   `EXCEPTION_CONTINUE_SEARCH` (-1) rethrow to the next wrapped frame; run `__finally`s while
   unwinding. (The scope handler/filter addresses are already in `SehExceptionInfo`.)
2. **Runtime throw:** `RtlUnwind` already raises (`seh_raise_guest_unwind`, patch 0007).
   Make **`sub_8242EA70` (`RtlRestoreContext`) also throw** the same guest-unwind so it
   propagates to `sub_82456198`'s host `SEH_CATCH` instead of returning to its caller. (Set
   it up as a runtime override, or special-case in codegen; do NOT use `longjmp_address`.)
   Hardware faults are already caught by `seh_filter`.
3. **`RtlCaptureContext` fill** (runtime): write the live `PPCContext` into `buf` (layout
   above; `thread->thread_state()->context()` + `TranslateVirtual`) so the game's dispatch
   and the `__except` see a valid context.
This keeps the non-local jump **inside the C++ function** (host stack unwinds to the
`__try`-owner's `catch`, which reaches its `__except`) — no external mid-body entry.

### Precise implementation breakdown (turnkey, ~multi-day; SehScope in `function_types.h`)
`SehScope = {tryStart, tryEnd, handler, filter}` (filter==0 ⇒ __finally, else __except).
The structure that works (goto OUT of a host `__except` to an enclosing label IS legal):
1. **Analysis phase (NOT codegen):** add each `__except` scope's `handler` PC to the
   function's `labels_` so a `loc_<handler>:` block is emitted + reachable. Blocks are
   formed from `labels_` during analysis (`FunctionNode::addLabel`), so adding at codegen
   time is too late. Do it where `SehExceptionInfo` is parsed (phase_register.cpp /
   parseSehScopeTable) or in a pre-codegen pass: `for s in scopes if s.filter: addLabel(s.handler)`.
2. **Codegen (`function_graph.cpp`):** emit `__seh_restart:` immediately before `SEH_TRY {`,
   and at the top of the try body a resume-dispatch:
   `if (rex_seh_resume()) { u32 rp = rex_seh_resume(); rex_seh_resume()=0; switch(rp){ case 0x<handler>: goto loc_<handler>; ... } }`.
3. **CATCH (`function_graph.cpp`):** replace the recover-to-caller with: restore the entry
   frame (as now), then for the relevant `__except` scope set `rex_seh_resume() = s.handler;
   goto __seh_restart;` (this leaves the host `__except`, re-enters the try, dispatch jumps
   to the handler in a fresh SEH scope). Keep recover-to-caller as the fallback for
   __finally-only functions / no matching scope.
4. **Runtime:** add a `thread_local uint32_t& rex_seh_resume()` accessor (seh.h/seh_win.cpp
   or init_h.inja).
5. **Refinements (correctness):** run the scope **filter** (call the guest filter at
   `s.filter` with the exception code/info) to choose EXECUTE_HANDLER vs CONTINUE_SEARCH
   (rethrow to the next wrapped frame), and set up the `__except`'s expected ctx
   (GetExceptionCode/Information). Start without the filter (assume EXECUTE_HANDLER, first
   `__except`) to validate the structure, then add the filter.
Build cycle per iteration: SDK build (function_graph.cpp + runtime) → `rexglue -f codegen`
→ `fix_recomp_labels.py` → app rebuild. Keep the committed stable baseline to revert to.

The sections below are the reasoning trail (the setjmp/longjmp "approach B" is historical).

## What the title actually uses (observed)
Standard **Win32 table-based SEH**, split between recompiled CRT code and kernel imports:
- `__imp__RtlCaptureContext` (×5 call sites) — captures the current CPU context into a
  caller-supplied buffer.
- `__imp__RtlUnwind` (×6) — unwinds frames to a target, running `__finally`s.
- `__imp____C_specific_handler` (×3) — the language handler the compiler calls per `__try`.
- `sub_8242EA70` = the game's **`RtlRestoreContext`** (recompiled): reloads a full saved
  context and jumps to it.

### The CONTEXT buffer layout (from `sub_8242EA70`'s loads; `r7 = r3 = buf`)
| offset | contents |
|---|---|
| `+0 .. +136` | `f14..f31` (18 doubles, `lfd`) |
| `+144` | saved **SP** (`r1`) |
| `+152 .. +296` | `r13..r31` (`ld`) |
| `+304` | saved **CR** |
| `+308` | **continuation PC** (restored into `lr`, then `blr`) |
| `+312` | mode flag (`==0` ⇒ restore-and-resume path; `!=0` ⇒ `RtlUnwind` path) |
| `+320 ..` | `v64..v127` (VMX) |

## Why it breaks in static recomp
`sub_8242EA70`'s resume path restores the context then `blr`s to `[buf+308]` — a **non-local
jump to a mid-function PC**. rexglue emits `blr` as C++ `return`, so it returns to its
*caller* carrying the restored (setjmp-time) `r1`/`r13..r31` → the caller continues with
`r31=0` → guest-null write → AV (caught by the SEH first-cut → worker dies → hang).

## Dead-ends ruled out (do NOT pursue these)
1. **`setjmp_address`/`longjmp_address` config.** rexglue supports it (`ppc_setjmp`/
   `ppc_longjmp`, host setjmp/longjmp keyed by the guest buffer address), **but there is no
   guest `setjmp`** — the context is captured by `RtlCaptureContext` (an import), not a
   matched guest save. Confirmed: no function stores `r13..r31` to a `[reg+152..296]` buffer.
2. **Host setjmp inside a runtime `RtlCaptureContext`.** `setjmp`'s frame must persist until
   `longjmp`; `RtlCaptureContext` *returns*, so its frame dies → restoring it is UB.
3. **Host `RtlCaptureContext`/`RtlRestoreContext` in the runtime stubs.** Same frame-death:
   the captured host context's frame is gone by restore time.
4. **C++-exception resume to a thread-entry dispatch loop.** Unwinds the host stack fine, but
   the resume target `[buf+308]` is **mid-function**; rexglue dispatches at function entry
   only, so it can't re-enter there.

The common wall is **mid-function entry** + **frame lifetime**. The only place a host
save can legally live is the frame that owns the `__try` (it persists across the try body).

## Recommended approach — inline `ppc_setjmp` at `RtlCaptureContext` call sites
Make rexglue emit a host `setjmp` *inline in the caller's frame* at each call to
`__imp__RtlCaptureContext`, and a host `longjmp` for `sub_8242EA70`. The caller is the
`__try`-owning frame, so it persists → no frame death; host longjmp resumes at the exact
host instruction after the (inlined) capture = mid-guest-function, safely.

Steps:
1. **Codegen:** special-case calls to `__imp__RtlCaptureContext` (in the import-call emitter,
   `src/codegen/builders/context.cpp` / `instruction_dispatch.cpp`) to emit, in the caller:
   `int rc = ppc_setjmp(ctx.r3.u32);` plus the PPCContext save/restore that the existing
   `setJmpAddress` path already does (context.cpp:179–184) **and** a guest-CONTEXT fill so
   the game's code that later reads `buf` (e.g. `[buf+308]`) still works.
2. **Config:** `longjmp_address = 0x8242EA70` → its body becomes `ppc_longjmp(r3, r4)`.
   (Note: this discards `sub_8242EA70`'s `RtlUnwind` path `[buf+312]!=0`; verify that path
   isn't needed for the boot, or keep it as a fallback.)
3. Regen → `fix_recomp_labels.py` → rebuild → test the worker no longer faults.

### Open risks to validate (why this is iterative, not a one-shot)
- **Semantic mismatch:** stock `ppc_setjmp`/`longjmp` restores the *setjmp-time* PPCContext
  snapshot, but `RtlRestoreContext` is supposed to restore **`buf`'s (handler-modified)**
  registers. If the handler changes registers (return value, SP), they'd be lost. Likely
  need a custom `ppc_longjmp` that restores from `buf` (`[buf+144]` SP, `[buf+152..296]`
  r13..r31, `[buf+0..136]` FPRs, `[buf+304]` CR) before the host `longjmp`.
- **`buf` must be filled** by the inlined capture (RtlCaptureContext's real job) so any guest
  code reading `buf` (PC/SP/regs) sees valid data.
- **`setjmp`-returns-twice:** the code after `RtlCaptureContext` re-runs on resume. This
  *matches* `RtlRestoreContext` semantics (it resumes at `[buf+308]` = right after capture),
  but confirm the CRT dispatch logic (`sub_82456198` and callers) tolerates it.

## Alternative (heavier) — full host-SEH exception dispatch
Implement `RtlUnwind`/`__C_specific_handler`/`RaiseException` in the runtime to drive host
C++ exceptions, and have `generate_exception_handlers` wrap each `__try`-owning function so
its host `catch` runs the guest `__except` (a label in the *same* function — no external
mid-function entry needed). This is the "correct" general model but a larger change and the
catch must run the real filter/handler (the first-cut's catch only recovers-to-caller).

## Validation
The worker faults at `sub_824711D0+0x5A1` (write to guest-null). Success = that fault stops
and the boot advances past the black-screen loading wait toward the menu. Diagnose with the
SEH fault backtrace (`seh_win.cpp`) + cdb-attach `~*k` + the window screenshot (see
[[35-entry-forensics]]). Keep the stable rendering build (committed) to revert to.
