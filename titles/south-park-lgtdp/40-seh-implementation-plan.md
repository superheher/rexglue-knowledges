# SEH implementation plan — the boot's last blocker (resume after exception)

This is the concrete plan to clear the **current** blocker: the recomp renders the first
frame, then a worker thread faults during asset load, the SEH first-cut *recovers* it
(killing the worker), and the main thread hangs waiting on it. The real fix is making the
guest's exception path **resume** correctly. Companion to [[35-entry-forensics]];
general lessons in `general/95`, `general/80`.

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
