# SEH implementation plan — the boot's last blocker (resume after exception)

## ✅✅ SOLVED + VERIFIED 2026-05-24 — it was a CUSTOM setjmp/longjmp, not Win32 SEH
The post-rendering hang is **fixed**; the recomp now boots to the **TITLE SCREEN**. The
blocker was a **hand-rolled C++ setjmp/longjmp** the game uses for **image-format detection**
(NOT `.xdata` table SEH — that whole framing, below, was a wrong turn). Verified by live
instrumentation:
- The image loader `sub_82459B00` iterates a format table and tries each decoder. **"Try JPEG"**
  `sub_82458010` does **`setjmp(buf = r1+720)` via `sub_8242EEA0`** (an EH-hook dispatcher that
  tail-calls the installed setjmp), runs the JPEG decoder, then `if (r3 != 0) goto fail`.
- The JPEG parser `sub_824711D0` checks the SOI marker; the asset is a **TGA** (`00 00 02 00`),
  so it calls its raiseError vtable method `sub_82456198`, which **`longjmp`s via `sub_8242EA70`**
  (the game's RtlRestoreContext) back to the setjmp → `sub_82458010` returns failure → the loader
  tries the next format.
- setjmp & longjmp use the **same buffer** (`vtable+144 == r1+720`, verified live `0x7048EA60`)
  and the setjmp frame is **alive** at longjmp time → rexglue's `ppc_setjmp/longjmp` models it
  exactly (its codegen snapshots/restores `ctx` around the host setjmp/longjmp).

**THE FIX (config-only, no SDK change):** manifest `[entrypoint]`:
`setjmp_address = 0x8242EEA0`, `longjmp_address = 0x8242EA70`; then regen + `fix_recomp_labels`
+ build. (The earlier failed attempt used `setjmp_address = 0x825925CC` = `RtlCaptureContext`,
a *different* buffer → `ppc_longjmp` no-match → abort. The real setjmp is the **EH-hook
dispatcher `sub_8242EEA0`**, found by tracing the loader, not the imports.) This also
supersedes the `fix_recomp_labels` Fix-3 band-aid for `sub_8242EEA0` (ppc_setjmp returns 0 on
first call = the same "run body" the band-aid forced).

**Follow-on:** the boot then hit `[FATAL] Unresolved call ... to 0x821F23EC` — a cross-function
`b` rexglue couldn't resolve. Fixed by registering all 18 such unresolved-branch targets in
`config/sp_functions.toml` (`gen_missing_funcs.py` KNOWN_COMPUTED) + making that script
cumulative. ⇒ **boots to the title screen.** Everything below is the (long) reasoning trail,
including the wrong "standard Win32 SEH dispatch" hypothesis — kept for the lessons.

---


This is the concrete plan to clear the **current** blocker: the recomp renders the first
frame, then a worker thread faults during asset load, the SEH first-cut *recovers* it
(killing the worker), and the main thread hangs waiting on it. The real fix is making the
guest's exception path **resume** correctly. Companion to [[35-entry-forensics]];
general lessons in `general/95`, `general/80`.

## 🔬 EMPIRICAL BREAKTHROUGH (2026-05-24, live instrumentation — supersedes the "standard SEH" framing)
Instrumented the runtime (`RtlCaptureContext`/`RtlUnwind`) **and** the generated
`sub_8242EA70`/`sub_82456198`/`sub_824711D0`, rebuilt, ran, read the log. Ground truth:

1. **The exception is LEGITIMATE — it is image-format detection via exceptions.**
   `sub_824711D0` is a **JPEG** marker parser: it checks the first two bytes for `FF D8` (SOI).
   The asset it's fed starts with **`00 00 02 00 00 00 00 00`** (a **TGA** header — `00 00 02`
   = uncompressed true-color; the game's assets are TGA/PNG, not JPEG — see `private/extracted`).
   So the JPEG parser **correctly** rejects it and calls its `raiseError` virtual
   (`[[parser+0]+0] = sub_82456198`) to throw → the loader is meant to **catch and try the next
   format**. This is on the **critical path for every non-JPEG image load** — there is no
   band-aid; the menu needs working image loads.
2. **The mechanism is a CUSTOM hand-rolled C++ exception runtime, NOT `.xdata` table SEH.**
   The parser object is **stack-allocated** (`obj=0x7048E7F0`) with a **stack vtable**
   (`P=[obj+0]=0x7048E9D0`): `[P+0]=sub_82456198` (raiseError), `[P+8]=sub_82137018` (a **no-op
   `blr`** cleanup), and an embedded **CONTEXT at `[P+144]`** (`0x7048EA60`) in the exact
   layout `sub_8242EA70` (RtlRestoreContext) reads. `sub_82456198` = run cleanup (`[P+8]`, no-op)
   then `RtlRestoreContext([P+144], 1)` to resume at the catch.
3. **`buf[308]` (the resume PC) is 0 — the CONTEXT is never captured in the recomp.**
   Logged at the live call: `buf=7048EA60 cont308=00000000 flag312=00000000 r4=1` → it takes the
   **resume path** (`flag312==0`) and `blr`s to **0**. `RtlCaptureContext` is called **0 times**
   before the crash; its only 2 guest call sites target a **global** (`sub_8243FCF0`) and the
   **throw-path stack buffer** (`sub_82446CE0`: capture→`RtlUnwind`, a separate `_CxxThrowException`-
   like path), **never** the parser object's `+144`. No guest fn captures a full CONTEXT into a
   passed buffer either. ⇒ whatever sets up the parser object's resume CONTEXT (the "try") is not
   running / not capturing in the recomp.
4. **No host-SEH catch can help.** Only **77** functions are `SEH_TRY`-wrapped (the `.xdata`
   scopes rexglue found); **none** of the loader-chain ancestors (`sub_82458010`, `sub_8246F498`,
   `sub_8246F270`, `sub_82476FD0`, `sub_824712B0`, `sub_824557A8`, `sub_82459B00`, `sub_82455F80`,
   `sub_8224D470`) are wrapped. A host `throw` from `sub_82456198` propagates to the **trampoline**
   (confirmed earlier). So the fix is **not** generate_exception_handlers + RtlUnwind.

**⇒ Revised fix path (custom EH, not Win32 dispatch):** find where the parser object / its stack
vtable is constructed (the loader's "try" — an ancestor that stores `sub_82456198` at `[P+0]`),
and ensure the resume CONTEXT at `[P+144]` is **captured there** (PC=the catch / "format failed,
try next"), then make `sub_8242EA70`'s resume `blr` **call the function at `buf[308]`** instead of
`return`. The capture is the linchpin (without it `buf[308]=0`). Diagnostics that proved this are
local edits to the runtime + git-ignored generated files (`[SEH-DIAG]` log lines).

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

### ATTEMPTED 2026-05-24 (host-SEH-CATCH dispatch) — blocked on missing handler labels (REVERTED)
Implemented pieces 2+3 (restart label + resume-dispatch + CATCH `__seh_resume=handler; goto
__seh_restart`) using a **function-local** `__seh_resume` (no TLS needed). The SDK + the
dispatch *structure* compiled, and the codegen emitted it for **33 functions**. BUT the app
build **failed widely** with `error: use of undeclared label 'loc_<handler>'` (recomp.13/17/
19/28/30/…). So **piece 1 is NOT actually satisfied**: `phase_register.cpp:657`
`addLabelToFunction(scope.handler)` adds the handler to `labels_`, yet the body emitter does
**not** always emit a `loc_<handler>:` — the handler PC isn't a reachable block boundary in
the emitted instruction stream (mid-block, or outside the function's emitted range). ⇒ Before
the catch can `goto` the handler, the **body/label emitter must emit a `loc_` at every
`__except` handler PC** (force a block boundary there; verify it's within the emitted range).
Also note: even with that, the exception must **propagate** — `sub_8242EA70`
(`RtlRestoreContext`) currently *returns-corrupted* (a normal return), so nothing reaches a
CATCH; it must **throw** (`seh_raise_guest_unwind`, e.g. a codegen special-case on its body)
for the dispatch to fire, AND the throw must land on the frame whose `__except == buf[308]`
(my first-cut jumps to the catching function's *first* `__except`, which needs the filter to
disambiguate). **ROOT of the label failure (confirmed):** the `__except` handlers are **MSVC SEH funclets**
— each `scope.handler` PC is emitted as a **separate `DEFINE_REX_FUNC(sub_<handler>)`
function** (verified: sub_8242B460, sub_82442384, sub_8230EEF8, sub_8229BAE4 all exist as
functions), NOT an in-parent label. So `goto loc_<handler>` can *never* work. ⇒ **The correct
host-SEH dispatch must CALL the funclet**: in the CATCH, restore the parent frame, then
`sub_<handler>(ctx, base);` (the funclet runs the `__except` body in the parent's frame), then
resume the parent after the `__try` (≈ `scope.tryEnd`). This replaces piece-1/2/3 above:
- CATCH: `ctx.r1.u32=__seh_r1; …; sub_<handler>(ctx, base); /* then continue after __try */`.
- The funclet calling convention (MSVC passes the parent frame pointer; the funclet may set a
  continuation) + the post-funclet resume (jump to after `__try`) are the intricate parts.
- Still need RtlRestoreContext(`sub_8242EA70`)→throw so the exception reaches the CATCH, and
  filter eval to pick the right scope when there are several.
Net: the correct approach is to **emulate the MSVC __except-funclet model** (call the funclet
+ resume after the try + filter) — genuinely multi-day. Reverted to the stable
recover-to-caller baseline (verified: town renders, no crash).

## ✅ DEFINITIVE FIX RECIPE (4th attempt proved the mechanism, 2026-05-24)
4th attempt: made `sub_8242EA70` (RtlRestoreContext) **throw** (`seh_raise_guest_unwind`)
instead of returning-corrupted. Result: the `sub_824711D0` corruption-fault **disappeared**
(SEHfault=0 — good), but the throw propagated to the **trampoline** `sub_82450FD0`, NOT a
wrapped `__try`-owner. **Proof that host-SEH-CATCH-in-an-ancestor can NEVER work:** by the
time `RtlRestoreContext` runs, `RtlUnwind` has already **unwound the `__try`-owner's frame
off the stack**, so its `__except` is not a host-stack ancestor — it's reached by a **full
context switch**, not stack propagation. `RtlRestoreContext`'s tail `blr` jumps to
`buf[308]` = the **handler funclet PC** (a separate `sub_<...>` function), which the recomp
emits as `return`.

**The fix is a runtime context-switch, 3 pieces (no host-SEH wrapping needed for this path):**
1. **`RtlCaptureContext` must FILL the buffer** so `buf[308]`/regs are valid (layout,
   big-endian): f14–f31@`+0..136`, SP(r1)@`+144`, r13–r31@`+152..296`, packed CR@`+304`,
   PC@`+308`=`ctx.lr`, VMX@`+320..`. **✅ IMPLEMENTED + verified-safe 2026-05-24**
   (`xboxkrnl_rtl.cpp` `RtlCaptureContext_entry`; was a no-op stub) — fills via
   `XThread::GetCurrentThread()->thread_state()->context()` + `TranslateVirtual` + `byte_swap`;
   GPRs/FPRs/SP/CR/PC filled, VMX deferred. Compiles, no rendering regression. This alone
   does NOT unblock (RtlUnwind still throws via the first-cut, so the dispatch never runs).
2. **`RtlRestoreContext` (`sub_8242EA70`) must CALL the funclet, not return.** Its resume
   path already reloads ctx from buf (incl. `ctx.lr=[buf+308]`, `ctx.r1=[buf+144]`); change
   the final `blr` (emitted as `return`) to **`REX_CALL_INDIRECT_FUNC(ctx.lr); return;`** so
   it tail-calls the handler funclet at `buf[308]` with the restored context. (Codegen
   special-case on `sub_8242EA70`, or a fix_recomp_labels rewrite of that blr.) Note the
   function has two paths — only the resume path (`[buf+312]==0`) gets this; the
   `[buf+312]!=0` path is the real `RtlUnwind`.
3. **Continuation:** after the funclet runs the `__except`, the parent resumes after its
   `__try`. The funclet (MSVC) yields the continuation; validate what `sub_<handler>` returns
   and where control should go (likely the funclet itself continues the parent). Start
   without this (just call the funclet) to see how far the worker gets.
Both #1 and #2 are required together (without #1, `buf[308]` is garbage → bad indirect call).
Each iteration: SDK build (RtlCaptureContext) + the sub_8242EA70 change + regen/fix + app
build. Keep the committed stable baseline (renders) to revert to.

**TESTED 2026-05-24 (pieces 1+2) → `RtlUnwind` is the missing critical piece.** With #1 done
and #2 (the funclet call) hand-edited in, the boot **FATALs `Call to … function at guest
address 0x00000000`** — i.e. **`buf[308]` is 0**. The game's recompiled CRT relies on
**`RtlUnwind`** to walk the `.xdata` scopes, run `__finally`s, and **write `buf[308]` = the
`__except` target** before `RtlRestoreContext` jumps there; our `RtlUnwind` is a first-cut
that just *throws*, so `buf[308]` is never set → 0. ⇒ the real ordering is **(2) implement
`RtlUnwind` = the core Win32 unwind** (walk `.pdata`/`.xdata` for the faulting PC, find the
handling scope via its filter, run intervening `__finally`s, set the target context incl.
`buf[308]`), *then* (3) `RtlRestoreContext`→call-funclet (piece "#2" above), then (4) the
continuation. `RtlUnwind` is the days+ gate. Pieces 1+2-without-RtlUnwind reverted to stable.

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
