# Entry-point forensics — why the boot stalls (definitive)

> ## ✅ FINAL — source-confirmed from the canary clone (2026-05-23)
> This block supersedes the `r3=-1` / "needs a cleaner trace" claims further down.
> Source = a full clone of xenia-canary at `~\xbla-refs\xenia-canary`.
>
> **The XEX entry `0x824499A0` is the common XapiThreadStartup trampoline** — run by
> *every* guest thread, not just main. That is why the aggregated `ftrace.0`
> per-instruction counts show **both** arms of its `r3==-1` test executing (different
> threads take different arms). It is *not* evidence that the main thread uses `r3=-1`.
>
> **Main thread launch (authoritative):** `KernelState::LaunchModule`
> (`kernel/kernel_state.cc:417`):
> `new XThread(ks, module->stack_size(), 0, module->entry_point(), 0, X_CREATE_SUSPENDED, …)`
> → `xapi_thread_startup = 0`, `start_address = 0x824499A0`, **`start_context = 0`**
> (so **r3 = 0**, not −1). `XThread::Execute` takes the *else* (raw) branch
> (`address=start_address`, `args=[start_context]`, `want_exit_code=true`). The reenter
> loop only fires on `FiberReentryException`/`longjmp` from `Reenter()`
> (`KeSetCurrentStackPointers`), which **South Park never calls** (absent from the
> boot log). **⇒ keep patch 0002 (start_context→−1) REVERTED — canary uses 0.** Either
> arm of the trampoline reaches the *same* epilogue and `blr`s to `[r1+0x68]`, so r3
> does not change the continuation anyway.
>
> **canary `Processor::Execute` == rexglue `FunctionDispatcher::Execute`** byte for
> byte (`ctx.r1 -= 64+112`; `ctx.lr = 0xBCBCBCBC`; call; restore). The args overload
> (`processor.cc:413`) only writes the stack when `arg_count > 7` — the 1-arg main
> thread writes **nothing**. The **only** functional difference between emulator and
> recomp is `blr`: canary's JIT follows it to `ctx.lr`; rexglue's static `build_blr`
> emits C++ `return;` (`codegen/builders/control_flow.cpp:94`) with no dispatch.
>
> **The airtight contradiction (the real open problem).** The trampoline's epilogue
> sets `ctx.lr = [r1+0x68] = [stack_base-0x68]` (`0x7018FFB8` on main) and `blr`s
> there. I checked *every* path that could populate that slot — `XThread::Create` /
> `AllocateStack` (only TLS is zeroed; guard pages at `stack_base`), `ThreadState`
> ctor (`r[1]=stack_base`), both `Processor::Execute` overloads — **none writes it**,
> and guest memory is zeroed on alloc (`memory.cc:699 Zero`). So the static model
> predicts `[r1+0x68]=0` → `blr 0` → `ResolveFunction(0)` fails → `Execute` returns →
> thread exits. **Yet canary reliably boots to the menu.** Something at runtime
> (heap-reuse leftover, or JIT indirect-branch handling of a 0/invalid target) supplies
> the continuation in a way static source-reading cannot reveal. In **stock** master
> the same slot is poison (`0xBE…`) → the documented `blr 0xBEBEBEBE` crash; canary
> differs only by not poisoning.
>
> **Recomp defect & fix.** `build_blr=return` discards the `[r1+0x68]` continuation.
> The fix needs (a) the **actual continuation value** and (b) a dispatch (write it onto
> the main-thread stack + reenter loop, or special-case the trampoline). (a) is the
> blocker and is **only obtainable live**.
>
> **NEXT (the chosen "build canary with tracing" path):** instrument the clone with a
> ~2-line log — print `ctx.lr` at the trampoline `blr` (`0x824499CC`), or log the first
> few `Processor::ResolveFunction` calls on the main thread — build canary, run the
> STFS package, read the continuation address from the log. That single value unblocks
> the recomp boot.

> ## ⚠️ CORRECTION (2026-05-23, later same day) — the title DOES boot in Xenia
> The earlier "does not boot in Xenia / research-grade" verdict in this file was
> **WRONG**, caused by a *setup error*: I tested the **loose extracted `default.xex`**.
> With the **proper STFS package** (`58410931/000D0000/…`), **Xenia canary fully
> boots South Park** — base `default.xex`, **no patch/title-update** (`PatchDB:
> Loaded patches for 0 titles`): Main XThread spawns ~14 game threads, loads content
> (`\media\Assets\Audio`, `\UI`, `\media\Assets\LuaScripts`, `strings`), inits audio,
> and **issues GPU draws** (menu). Xenia game-compatibility #1156 = **"state-menus"**
> (intro + main menu work; gameplay blocked by a save/profile error). So the boot is
> **reproducible in an emulator from the same base xex the recomp uses** — it is
> **not** research-grade. The static "stub entry returns immediately" reading below
> is therefore **incomplete**: entered correctly (kernel-set-up frame + content),
> `0x824499A0` runs the full game init. The recomp's early-return is a **fixable
> runtime/setup discrepancy** (likely loose-vs-STFS content mount and/or entry
> stack-frame setup), now being diagnosed against the working canary trace. The
> static tooling/findings below remain accurate as *static* facts; the *conclusions*
> about bootability are superseded by this correction. See [[90-progress-report]].

### BREAKTHROUGH — canary boot trace decodes the mechanism (2026-05-23)

Captured a **function-execution trace** of canary's boot: run with
`--trace_function_data=true --trace_function_references=true --trace_function_data_path=<dir>`,
boot the STFS package, then **close gracefully** (`CloseMainWindow()` — a force-kill
discards the trace). Output `ftrace.0` (Xenia `FunctionTraceData` format: per-function
header with `start/end/call_count/thread_use/caller_history[4]` + per-instruction
execute counts; parser `tools/parse_ftrace.py`). Findings:

- **Only ~339 functions execute** during boot-to-menu (huge reduction from 20,045).
- **The entry `0x824499A0` is the main thread's only `0xE0000000`-rooted function**
  (Xenia's thread-start sentinel). The whole boot runs under it.
- **Canary passes `r3 = -1`** to the entry, NOT `r3=0`: the instruction counts show the
  `bne` at `0x824499A4` was **not** taken — the `r3==-1` path ran (`li r3,0; bl EC28;
  …`). (So the long-reverted patch 0002 `r3=-1` matched *canary*; stock master used
  `r3=0`.) The entry still returns; `r3=-1` runs `EC28` which stores 0 to a KTHREAD
  field — likely required thread-state setup.
- The deeper boot (`main` → menu idle loop `0x82109BB0`, 154M calls) runs under the
  entry on the main thread.

> **⚠️ Caveat (retraction):** an earlier draft read the `caller_history[4]` values as
> a clean call graph and inferred "the boot iterates a C++ init array at
> `0x820DAxxx`." That was **wrong** — `caller_history` here is **noisy** (it mixes
> stack addresses `0x7018fbXX`, `.data`, and `.rdata` **data** addresses, e.g.
> `[0x820D0588]`=`"Down"`, `[0x820DACAC]`=UTF-16 data — *not* function-pointer
> slots). So the trace's caller history **cannot** be used to reconstruct the call
> graph or pin the init array. Don't trust it for that.

**What the trace reliably establishes:** (a) canary passes **`r3=-1`** to the entry;
(b) the boot is **~339 functions**; (c) per-instruction execute-counts give each
function's exact taken path. It does **not** reliably give the call graph /
continuation address.

**Fix leads:** (1) launch the main thread with `start_context = -1` (re-apply patch
0002 — canary-confirmed; *necessary but tested insufficient alone* — the entry still
returns). (2) The continuation (how the entry leads into the 339-function boot) is
still unresolved — `caller_history` is too noisy; needs a cleaner trace (a Xenia
build with proper call-trace, or stepping the entry's return in a debugger) to read
`[r1+0x68]`. The 339-function set is a useful boot-path reference regardless.

### Source-confirmed defect + fix (the precise reason the recomp early-returns)

Reading the runtimes' source (not inference) pins it:

- **Xenia** `Processor::Execute` sets `ctx->lr = 0xBCBCBCBC` (a sentinel) then runs the
  entry. A *normal* entry's prologue saves that sentinel LR and its epilogue restores
  it → `blr` to the sentinel → the JIT returns (Execute ends). **This title's entry is
  mid-function**, so the prologue is skipped and the epilogue does `lwz r12,-8(r1);
  mtlr r12; blr` — i.e. it `blr`s to **`[r1+0x68]` (a stack value), not the
  sentinel** — and Xenia's JIT *continues executing there* (the boot continuation;
  poison→crash in stock, valid→boot in canary).
- **rexglue** (static recomp) recompiles `blr` as a plain **`return;`**
  (`src/codegen/builders/control_flow.cpp` `build_blr`). So the recompiled entry's
  `blr` returns to the C++ runtime (`FunctionDispatcher::Execute` → thread ends),
  **discarding `ctx.lr` (= `[r1+0x68]`, the continuation)**. rexglue has **no reenter
  loop** (Xenia's `XThread::Execute` has one). *This is the exact reason the recomp
  prints "Execution complete" and stops.*

**Fix (two parts):**
1. **Reenter loop:** after the entry's recompiled function returns, if `ctx.lr` is a
   valid registered guest function (≠ sentinel `0xBCBCBCBC`), dispatch to it and
   repeat — emulating the interpreter's `blr`-follows-LR. (Add to the main-thread
   launch / `FunctionDispatcher`.)
2. **Valid `[r1+0x68]` continuation:** the recompiled entry's epilogue loads the
   continuation from the guest stack at `[r1+0x68]`, which rexglue currently leaves
   uninitialised. The kernel/canary writes the boot continuation there; rexglue must
   too. **The continuation address is the one remaining unknown** — being read from
   canary's thread-startup source (clone in progress) or via a debugger read of
   `[r1+0x68]`.

Also: launch with **`start_context = -1`** (canary value; the entry takes the `r3==-1`
EC28 path that stores 0 to a KTHREAD field).

**Source-confirmed (canary clone):** canary's `Processor::Execute` is *byte-identical*
to rexglue's `FunctionDispatcher::Execute` — both do `ctx.r1 -= 64+112`,
`ctx.lr = 0xBCBCBCBC`, call, restore. So the launch is the same; the **only**
difference is `blr`: canary's JIT continues at `ctx.lr`, rexglue's static `build_blr`
returns. The entry's epilogue sets `ctx.lr = [r1+0x68] = [stack_base-0x68]`
(`0x7018FFB8` for the main thread). In **stock** Xenia that stack slot is poison
(`Fill 0xBE`) → `blr 0xBEBEBEBE` crash (matches the stock crash dump exactly). In
**canary** it is *not* poisoned (canary zeroes only TLS, not the stack) and the boot
proceeds — so `[stack_base-0x68]` holds a **valid continuation**, but canary's source
neither poisons nor explicitly writes it (`ThreadState` sets `r1=stack_base`;
`Execute` only pads r1). `KeSetCurrentStackPointers`/`Reenter` is **not** used by
South Park (a Forza-path; absent from the boot log).
**The last unknown is the value at `[stack_base-0x68]` and how canary reliably has
it** — resolving it needs reading canary's *live* main-thread stack at the entry
(debugger / instrumented canary build), which is the next deep step. Once known: set
that continuation on the recomp's main-thread stack + add a reenter loop (dispatch to
`ctx.lr` after the entry) + `start_context=-1`.

### Corrected diagnosis + concrete next step (the recomp IS close)

What canary's boot proves about the mechanism:
- Canary's **Main XThread** (the XEX entry `0x824499A0`) runs the **full boot** —
  spawns the game's worker threads via `ExCreateThread`, loads content, inits audio,
  draws. So entered *correctly*, the entry leads straight into the game init.
- **Why stock Xenia crashes but canary boots:** Xenia's `XThread` fills the new
  stack with `0xBE` poison (`xthread.cc:249`). The mid-function entry skips the
  prologue that would save LR, so its epilogue `lwz r12,-8(r1); mtlr; blr` returns to
  `[r1-8]`. In stock that slot is poison → `blr 0xBEBEBEBE` crash. **Canary diverged**
  and boots the same xex (no patch) — so canary sets that return slot (or the thread
  start) to a valid **boot continuation**.
- **The recomp's gap:** rexglue sets the entry's return to a clean **thread-exit**
  trampoline (so the entry "returns" → `Execution complete` → thread ends, no game
  init). It must instead replicate the kernel/canary thread-startup so the entry
  continues into the boot.
- Content is NOT the blocker: `private/extracted/` has the paths the game uses
  (`media/Assets/Audio`, `UI`, `LuaScripts`, `strings`; 1555 files).

**Mechanism (precise) + what was tried:**
- In **static recomp**, a guest `blr` becomes a C++ `return`: rexglue
  (`function_dispatcher.cpp`) sets `lr=0xBCBCBCBC`, calls the recompiled entry as a
  C++ function, and on its `blr`/return drops back to the runtime → the thread ends
  ("Execution complete"). The mid-function entry's epilogue restores LR from
  **`[r1+0x68]`** and returns there — but rexglue never writes a boot continuation to
  that slot, and even if it did, the recompiled `blr` C++-returns instead of
  dispatching to it. Canary (interpreter) *follows* `blr` to `[r1+0x68]` and keeps
  executing — the kernel-set continuation. **That continuation is the missing piece.**
- **Tested via `REX_ENTRY_OVERRIDE` (verify-by-running):** entry `0x824499A0` →
  returns ("Execution complete"); entry `0x82449968` (function start, runs the
  skipped prologue + the `EC98` indirect kernel call via `[0x8260E0F0]+0x20`) →
  exits silently (the dispatch-table global isn't set up in rexglue). **Neither
  boots.** So it's not a wrong-entry-address issue; it's the thread-startup
  continuation/state the kernel provides around the entry.

**Concrete next step:** diff **Xenia *canary*'s** thread-launch / entry handling
(open source: `github.com/xenia-canary/xenia-canary`, `XThread::Execute` /
`PrepareThreadStartContext` / stack setup) against master and against rexglue's
`thread_state.cpp` — find what canary puts at the entry's return / how it drives the
main thread, and replicate it in rexglue (a launch patch). Then re-run with the
`REX_ENTRY_OVERRIDE` harness off (original entry) and verify game threads spawn. The
working canary trace (`xbla-refs/xenia-bin/canary_stfs.log`) is the oracle.

This is the deep, tool-backed analysis of South Park's executable entry, refining
[[30-boot-log]]. Everything here was produced **statically** from the decrypted PE
image (`tools/xex_decrypt.py --save`, then `tools/pe_inspect.py`, `tools/pdata.py`,
`tools/ppc_dis.py`, `tools/callgraph.py`, `tools/find_initarray.py`). Read-only;
no game code/assets are committed.

## The entry point is triple-confirmed and is *mid-function*

| Source | Value |
|--------|-------|
| XEX optional header `ENTRY_POINT` (0x00010100) | `0x824499A0` |
| PE `AddressOfEntryPoint` (RVA 0x4499A0 + ImageBase 0x82000000) | `0x824499A0` |
| Independent AES decrypt + decode (`tools/xex_decrypt.py`) | bytes match |

Image base `0x82000000`; `.text` `0x82100000`; machine **`0x01F2`
(POWERPCBE)**. So the entry value is not a derivation error.

But `0x824499A0` is **+0x38 inside the function that starts at `0x82449968`**
(clean prologue at `0x82449968`: `mflr r12; stw r12,-8; std r31,-0x10;
stwu r1,-0x70`). Full disassembly of that function:

```
82449968  mflr r12; stw r12,-8(r1); std r31,-0x10(r1); stwu r1,-0x70(r1)  ; PROLOGUE
82449978  mr r31,r4 ; addi r4,r1,0x50
82449980  bl 0x8244EC98            ; kernel-query helper (indirect via table @0x8260E0F0[+0x20])
82449984  cmpwi r3,0 ; beq 0x824499B8
8244998C  cmplwi cr6,r31,0 ; beq cr6,0x8244999C
82449994  lwz r11,0x50(r1) ; stw r11,0(r31)
8244999C  lwz r3,0x54(r1)
;; ====== XEX ENTRY POINT 0x824499A0 ======
824499A0  cmpwi cr6,r3,-1 ; bne cr6,0x824499BC     ; r3 != -1 -> epilogue
824499A8  li r3,0 ; bl 0x8244EC20 (-> 0x8244EC28) ; r3 == -1 path
824499B0  lwz r3,0x54(r1) ; b 0x824499BC
824499B8  li r3,-1
824499BC  addi r1,r1,0x70 ; lwz r12,-8(r1); mtlr r12; ld r31,-0x10(r1); blr  ; EPILOGUE
```

`0x8244EC20 -> 0x8244EC28` is also trivial:
```
8244EC28  lwz r11,0x150(r13) ; if !=0 return        ; r13 = PCR/TLS base
          lwz r11,0x100(r13) ; stw r3,0x160(r11)    ; store 0 into a thread field
          blr
```

## Why this cannot be the cold execution start (even on real HW)

Entering at `0x824499A0` **skips the prologue** (`stwu r1,-0x70` at `0x82449974`).
So no 0x70 frame is pushed, yet the epilogue runs `addi r1,r1,0x70` — which
**corrupts `r1`** (shifts it into the caller/guard region) and then restores
`r12`/`r31` from `[r1-8]/[r1-0x10]` (garbage). Concretely, called with `r3 = 0`
(the value Xenia and rexglue pass — confirmed in [[30-boot-log]]) it takes
`bne -> epilogue` and returns immediately having run **zero** game/CRT code; called
with `r3 = -1` it stores 0 to a TLS field and returns. **Neither path boots
anything, and the cold-entry frame math is broken** — so the real console is *not*
jumping to `0x824499A0` cold either. The recompiler/runtime model "create a thread
at the XEX entry and call it" is therefore insufficient for this title.

## What it is *not* (ruled out, with evidence)

- **Not a TLS-callback init.** `tools/pe_inspect.py`: the PE has **no TLS data
  directory**. (The XEX `TLS_INFO` header only sizes TLS slots; it is not a
  callback list.) So nothing runs via TLS callbacks before the entry.
- **Not an `r3 == -1` trick.** The `r3 == -1` path is also trivial (TLS store),
  so the reverted patch 0002 would not have booted it. [[30-boot-log]].
- **Not findable by a static direct-call graph.** `tools/callgraph.py` over the
  authoritative `.pdata` function list (10,672 funcs): the code is
  **indirect-call-dominated** (C++ vtables / function-pointer tables — see the
  3,045-entry vtable run at `0x820DBC7C`). Every `_initterm`-shaped function shows
  **0 direct callers**, so `mainCRTStartup`/`main` cannot be reached by walking
  `bl` edges. `find_initarray.py` finds vtables, not a small CRT init array.

## Cross-check with Ghidra (headless, PowerPC:BE:64)

Imported the decrypted image into **Ghidra 11.4.2** headless (raw binary, base
`0x82000000`, `.pdata` starts pre-defined; `tools/ghidra_pre_funcs.py` +
`tools/ghidra_find_main.py`). Result **confirms** the static finding and adds
nothing that contradicts it:

- `FUN_824499a0` (the entry) has **0 callers** and **callees = {`8244ec20`}** —
  i.e. Ghidra agrees the entry only reaches the trivial TLS-store stub.
- The "small functions called by the most roots" are `8242ce98`/`8242ce9c`
  (387/435 root callers) = `__savegprlr`/`__restgprlr` (GPR save/restore helpers),
  **not** `_initterm`.
- The call graph stays **indirect-dominated** even under Ghidra's analysis: the
  top "roots" by out-degree (`FUN_82376078` out=33, etc.) are **game-logic
  fragments** (they start with `lwz rX,0xNNN(r31)` on an already-set base pointer —
  no prologue), reached only by fall-through/branch, not `bl`. So a clean
  `mainCRTStartup`/`main` does **not** fall out of the static call graph.
  (Lesson: feeding imperfect `.pdata` starts as functions can split functions and
  pollute the graph — let Ghidra auto-find functions, or use clean boundaries.)

So three independent tools (capstone, `.pdata`, Ghidra) agree: the entry is a stub,
and the real boot trigger is **not statically reachable from the title** — it is
kernel/loader behaviour. A **dynamic boot trace is now the decisive next step.**

## "One more shot": hunting `mainCRTStartup` to start there — defeated headlessly

The plan was to find the real `mainCRTStartup` and configure the runtime to start
there (the runtime override is ready: a `REX_ENTRY_OVERRIDE` env-var hook for
`user_module.cpp` after it reads `XEX_HEADER_ENTRY_POINT`). Finding it failed via
**every** headless method tried — this binary is exceptionally hostile to static
analysis:

| Method (tool) | Why it failed here |
|---|---|
| Direct + resolved call graph (`callgraph`, Ghidra `getCalledFunctions`) | Calls are overwhelmingly **indirect (vtables/singletons)**; `_initterm`/`main` have 0 direct callers; thousands of "roots". |
| `_initterm` pattern: `bctrl`+`addi r,r,4`+`cmplw` (`find_crt initterm`) | Matches **any pointer-array loop** — 127 hits, all noise. |
| "calls 2+ `_initterm`-shaped" (`find_crt maincrt`) | Hits **vtable-dispatch loops**, returns mid-function fragments. |
| Clean Ghidra decompiler, top roots by out-degree (`ghidra_dump_roots`) | Top roots are **game logic**; Ghidra found only 7.5k funcs (missed many) and `mainCRTStartup` (low out-degree, kernel-called) didn't surface. |
| Init-array bounds `lis`+`addi` forming a `.text`-pointer run (`find_crt_initarray`) | Matches the title's **many C++ constructors/vtable setups** (e.g. `0x82104328` is a ctor storing vtables), not `.CRT$XC`. |
| `__security_cookie` = global with 1 writer/many readers (`find_security_cookie`) | Top hit `0x828F2D2C` (736 readers) is a **game singleton pointer**, not the cookie; its writer is a singleton accessor. |

Even capstone can't linearly decode `.text` (VMX128 / data-in-code; needs
`skipdata`). **Net: `mainCRTStartup` cannot be reliably pinned with headless
static analysis of this title.** And there is a deeper paradox: the XEX entry is a
*stub*, nothing calls it, and it doesn't call `mainCRTStartup` — so in the normal
flow `mainCRTStartup` is **never invoked** (yet the game shipped). That can only be
reconciled by a boot mechanism outside "call the XEX entry," which neither Xenia
version reproduces. Cracking it needs **interactive Ghidra/IDA** (a human-driven
decompiler session — the maintainer has Ghidra + RE expertise) or a **real-hardware
boot trace**; both are beyond headless autonomous static analysis.

## Empirical "force the entry" experiment — also negative (verify by running)

Static `mainCRTStartup` discovery failed, so we tested it **empirically**: an
env-var entry override (`REX_ENTRY_OVERRIDE`, patch `0004`) + a brute-force over the
**65 zero-reference, prologue-having candidates** (`tools/find_zeroref_roots.py` —
functions called only by the kernel, the pool that must contain `mainCRTStartup`).
For each: start the recomp's main thread there and watch the run log vs the stub
baseline (36 lines, ends "Execution complete"). Driven from a logged-on session via
`schtasks /it` (the runtime needs a desktop for its D3D12 window). Result:

| Outcome | Count | Meaning |
|---|---|---|
| Returned like the stub ("Execution complete") | 6 | not the CRT entry |
| Crashed on a garbage/indirect pointer (`Call to invalid … at 0x00000000/0x6F727452/…`) | 14 | random function run on `r3=0` garbage args |
| Ran a few seconds then exited, **never reaching the game phase** | ~44 | did some work, then returned/crashed silently |

**No candidate booted the game.** Across multiple signal passes (default log, trace
+noisy log, run-length/exit), **none** progressed past the runtime's pre-launch
("Translated 0 shaders") into the game phase — no game worker threads, no shader
translation, no present. (Trace logging didn't help: guest kernel-import calls
aren't logged even at trace, so the discriminator was reaching the game phase, which
none did.)

**Interpretation:** forcing any single function as the entry does **not** reproduce
this title's boot. Combined with "doesn't boot in Xenia" and the stub entry, this is
strong evidence the boot is **kernel-orchestrated** — it depends on state/sequence
the kernel sets up around the entry that "call function X on a bare thread" does not
satisfy. So even the *right* `mainCRTStartup` likely wouldn't boot when forced
in isolation. This closes the autonomous, headless attack surface.

## Conclusion / where the unblock must come from

`mainCRTStartup -> _initterm -> main` exists in the image (the game runs on HW) but
is reached by a path **not** captured by "call the XEX entry," and that path is
invisible to lightweight static analysis of *this* title (it is kernel/loader
behaviour and/or hidden behind indirect calls). The two viable unblocks:

1. **Dynamic boot trace** (Xenia with CPU/kernel logging, or real-HW) — shows the
   first guest addresses actually executed and how the entry is invoked. Definitive.
2. **Full decompiler reconstruction** (Ghidra headless, PPC BE) — recovers the
   indirect call graph and locates `main`; then the runtime entry can be overridden
   to `mainCRTStartup` and the boot re-tried empirically.

## Dynamic trace via Xenia — RESULT: it crashes identically (analysis vindicated)

**Captured a boot trace in stock Xenia (`xenia-project` master `v1.0.2844`, run via
a `schtasks /it` interactive task — see setup notes below).** Stock Xenia's CLI
launch works (canary's did not). The result is decisive and confirms the static
analysis *exactly*:

- Xenia reads `XEX_HEADER_ENTRY_POINT: 824499A0` (matches us) and creates
  **"Main XThread" (thid 6)** at the entry.
- The main thread goes **straight to a CRASH DUMP** — nothing runs between
  `XThread::Execute` and the crash (no imports, no init):
  - **`PC = 0x824499CC`** (the `blr` at the end of the entry stub)
  - **`r12 = 0xBEBEBEBE`, `r31 = 0xBEBEBEBEBEBEBE`** (Xenia's *poison* for
    uninitialised memory), `r1 = 0x7018FFC0`, all else 0.

That is precisely the predicted failure: entered cold at `0x824499A0`, the prologue
(`mflr r12; stw r12,-8(r1)` at `0x82449968`) is **skipped**, so the epilogue
`lwz r12,-8(r1); mtlr r12; … blr` restores a **poison return address** and jumps to
`0xBEBEBEBE`. **Stock Xenia crashes South Park the same way a naïve recomp would** —
the recomp is behaving correctly; the title's entry is the problem.

### What this means

- **The title does not boot via the standard "call the XEX entry" model** — proven
  in the reference emulator, not just inferred. South Park LGTDP is a genuinely hard
  title (consistent with no public recomp existing).
- **Root cause of the crash:** the XEX entry points *into* a function, past the
  prologue that saves LR. A normal entry (`mainCRTStartup`) saves Xenia's
  return-LR in its prologue and restores it; this entry never saves it, so the
  epilogue returns to garbage. On real HW the kernel's thread-startup must seed a
  valid return address at `[r1+0x68]` (the slot the epilogue reads) for this to
  return cleanly — a kernel-ABI detail stock Xenia doesn't replicate.
- **But fixing the crash ≠ booting:** even returning cleanly, the entry is a
  do-nothing stub (kernel-query + TLS store). The game's real init is reached by
  some other path. So the remaining unknown is unchanged — *how is the real init
  triggered* — and it is **not** answered by stock Xenia (which simply can't run
  this title).
### Canary too: no crash, but still no boot (decisive)

Also traced **Xenia canary** (`canary_experimental@09dbe2c`, run the same way; the
earlier canary "failures" were a *corrupted config* from my edits — a **fresh**
config + CLI flags works). Canary **does not crash** (it seeds a valid return frame
so the stub epilogue returns cleanly) — **but the game still never initializes**:

- Main XThread starts (`XThread::Execute thid 6`) and then logs **nothing more** at
  debug level — i.e. it makes **zero kernel calls** (the stub calls none) and the
  log is **static after ~20 s** (sampled at 20 s and 45 s: identical size). No game
  worker threads, no GPU draws/present.
- **206 "import variable was not resolved" warnings** (xboxkrnl/xam data imports
  Xenia can't bind) — a large, unusual unresolved-data surface for this title.

**Definitive conclusion:** South Park LGTDP **does not boot in Xenia** (stock
*or* canary). In both, guest execution is **only the XEX entry stub** — stock dies
in its epilogue, canary returns from it and the title goes no further. The
recompilation **mirrors the reference emulator exactly**, so the recomp is correct;
the title's real boot is **not supported by the reference emulator**, which means
making it *playable* is **research-grade** (needs real-HW boot tracing or deep
kernel RE to discover how the real init is triggered past the stub). This is a
**feasibility finding** that the import-only [[00-feasibility]] analysis could not
have predicted — the entry/boot anomaly, not the import surface, is the blocker.

## Dynamic trace via Xenia — setup notes

Set up **Xenia canary** as a boot-trace oracle (download `xenia_canary_windows.zip`
from `xenia-canary/xenia-canary` releases). Config for a clean, max-detail boot log
(`xenia-canary.config.toml`):

- `[Content] license_mask = 1` — **required** for XBLA titles to pass the license
  check; with the default `0` the title load never even starts.
- `[General] discord = false` — `discord = true` makes `DiscordPresence::Initialize()`
  **hang** at startup (right after the "Cache root" log) when no Discord client /
  in an automated session.
- `[Logging] log_level = 3` (debug), `log_mask = 0` (all categories),
  `log_file = "...xenia_boot.log"`, `flush_log = true`.
- `[GPU] gpu = "d3d12"` (or `null` for CPU-only), `[APU] apu = "nop"`.

**Gotcha — can't drive it fully headless from automation:** in a non-interactive
shell context Xenia hangs at `EmulatorWindow::Create()` (window needs an interactive
desktop; CPU ~0%, ~28 MB, won't progress). Worse, in this build the **CLI title
launch did not fire** by any form tried (positional, `--target=`, `-- <path>`, for
both a loose `.xex` and the STFS package) — Xenia opens its library window and idles
with `cvars::target` empty (no "Loading module" / "Failed to launch target" logged).
So the title must be opened via the **GUI** (drag-drop / File-Open) in an interactive
session. That one GUI action is the maintainer hand-off point.

Once a log is captured it should show module load, **main-thread creation with its
entry address**, the kernel-call sequence, and whether South Park reaches a frame in
Xenia — which will finally reveal how the real boot is triggered vs. the stub entry.

Both are larger efforts; this is the genuine multi-week core of bring-up the
feasibility note ([[00-feasibility]]) predicted. Up to this point — extract →
recompile → build → boot the runtime → execute guest code → fully characterise the
entry — is **done and verified**.
