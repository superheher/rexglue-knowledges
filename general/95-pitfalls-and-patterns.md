# Pitfalls & patterns — a quick-reference catalog

Recurring problems and their fixes, grouped by area. This is the companion to the
per-subsystem deep dives; promote new, general findings here from title case
studies so the catalog compounds.

Format: **Symptom → Cause → Fix** (with the deep-dive doc in parentheses).

## Build / toolchain
- **Build fails with MSVC/GCC errors or weird codegen.** → Wrong compiler. →
  Use **Clang 18–20+**; the recompilers and their output rely on Clang
  intrinsics/codegen. (`30`)
- **Submodule headers missing at build.** → Nested submodules not initialized. →
  `git submodule update --init --recursive`. (`30`)
- **rexglue API/codegen changed after a bump.** → It's early-development. → Pin
  commits; bump deliberately; re-test. (`30`)

## Extraction / recon
- **Extractor produces a corrupt XEX.** → Block→offset math ignored interleaved
  **hash blocks**. → Skip a level-0 hash block every `0xAA` data blocks; data
  starts at `0xC000`, 4 KiB blocks. (`25`)
- **Addresses/jump tables don't match the running game.** → A **Title Update**
  changes code; you recompiled the base. → Apply the `.xexp` and recompile the
  **patched** XEX. (`20`)
- **Parsing guest structures (`.pdata`, vtables, pointer arrays) yields garbage /
  ASCII / `0xFFFFFFFF`.** → **Endianness**: PE *headers* are little-endian but all
  *guest content* is **big-endian**. → Read guest tables big-endian. (`45`)
- **The `.pdata`/EXCEPTION table isn't where its dir RVA says, or one record spans
  several functions.** → In a basic-decompressed image the dir RVA can be a page
  off; and `.pdata` **merges adjacent tiny frame-sharing functions**. →
  Auto-locate the table by signature; treat a record as *≥1* function and confirm
  small boundaries via prologues. `.pdata` starts are still the best authoritative
  **function-boundary** source to feed the recompiler. (`45`)
- **Tools won't disassemble the title PE ("unknown arch").** → Machine type is
  **`0x01F2` POWERPCBE**, which `llvm-objdump`/most PE tools don't handle. → Use
  **capstone** PPC big-endian 32-bit (`file_off = addr - image_base`). (`45`)

## CPU structural
- **Trap/`__builtin_trap`/crash at an indirect branch.** → Missing **jump table**
  entry. → Add it via `XenonAnalyse`/scanner output or hand-author in the switch
  table TOML. (`50`)
- **`Call to invalid or unregistered function at 0x...` at runtime (an indirect
  `bctr`/`bctrl`).** → The analyzer never emitted/registered a function at that
  address, so the dispatcher can't resolve it. Two classes: **(a)** a **vtable
  method** in a vtable the scanner didn't recognise, or **(b)** a **computed-jump /
  adjustor-thunk target inside a larger function** (reached via a runtime-computed
  `ctr`; it is NOT a static pointer, so a `.data` pointer scan won't find it — it only
  surfaces at runtime). → Register the address(es) as **CONFIG functions** (`[functions]`
  table, `"0xADDR" = {}` — empty ⇒ extent auto-discovered; CONFIG authority means it
  won't be merged away even mid-function), layered into the entrypoint via `includes`,
  then re-codegen. Find the static-vtable class by scanning the image for runs of ≥3
  consecutive 4-aligned code pointers minus the registered set; add class (b) as each
  one FATALs. Registering a mid-function target works because it executes that tail and
  returns. (`50`, `45`)
- **A function bleeds into the next; bad returns/stack.** → Wrong **function
  boundary** or wrong **register save/restore** address. → Declare boundaries in
  `functions`; fix save/restore addresses by byte pattern. (`50`)
- **Crash entering a routine that unwinds.** → `longjmp`/`setjmp` not redirected,
  or **EH data** parsed as code. → Set their addresses; add `invalid_instructions`
  skips. (`50`)
- **Nonvolatile register / stack pointer corrupted across a call** (caller's `rNN`
  or `r1` is garbage after a callee returns; manifests as a null/`base+0` write or a
  read near 0 *deep in the boot, ~tens of seconds in*). → The **callee was emitted
  without an epilogue**: in the binary it ends with `bl <helper>` then padding
  (`0x00000000`), with no `addi r1`/`ld rNN`/`blr`. On hardware the helper does the
  tail-cleanup or never returns (a `longjmp`/`noreturn`); the recompiler translated the
  `bl` as a plain call + a void return, so `r1`/nonvolatiles are never restored and the
  guest stack pointer drifts toward 0. → It's the title's **exception/control-transfer**
  path. Common cause: **Win32 SEH** — a function calls `RtlUnwind` (noreturn on HW: unwinds
  to a handler), but the runtime stubs `RtlUnwind`/`__C_specific_handler` as no-ops that
  *return*, and SEH-handler generation is off. Fix: enable the recompiler's SEH-handler
  generation **and** implement `RtlUnwind`/`__C_specific_handler`/`RtlRaiseException` to
  drive host `__try/__except` (or a C++ exception) so unwinds reach the right handler;
  for `setjmp`/`longjmp`-based titles, set their addresses instead. SEH/longjmp is one of
  the hardest parts of static recomp — expect to implement, not just configure. Bisect with
  a `REXLOG_WARN` of the suspect register before/after each indirect call. (`50`, `80`)
- **Catching a guest fault stops the crash but doesn't make the work succeed (recover ≠
  resume).** An SEH "first cut" that wraps functions in host `__try`/`__except` and, on
  catch, restores the entry frame + returns failure to the caller **prevents the process
  death** but **kills the faulting guest operation** — if that operation is a worker the
  rest of the boot waits on, you trade a crash for a **hang** (the waiter never gets its
  completion signal). The real fix is to *resume* the guest's own handler. Watch for the
  title's **`RtlRestoreContext`/`longjmp`** — a function that reloads a saved register
  context (f14–f31/r13–r31/VMX, SP, and a continuation PC from a buffer) then `blr`s to it:
  static recomp emits that `blr` as a C++ `return`, so it returns to the **caller** carrying
  the *restored* (setjmp-time) registers → corruption a few instructions later. It's a
  **non-local jump to a mid-function PC**. If the title uses table-based SEH
  (`RtlCaptureContext`/`RtlUnwind`/`__C_specific_handler` imports) there is **no guest
  `setjmp`**, so the `setjmp_address`/`longjmp_address` shortcut does **not** apply — you
  must implement the exception dispatch + mid-function resume. (`50`, `80`)
- **Works in Debug, breaks with optimizations.** → An `*_as_local`/`skip_lr`
  assumption (clean ABI / no exceptions) is violated. → Disable the offending
  optimization; only enable opts after a stable boot. (`50`)

## Runtime launch / first guest execution
- **Links, but "No function registered at \<entry\>" (nothing registers).** → The
  recompiler emits a function-mapping table that the runtime walks until
  `guest == 0`, but a legitimate **address-0 entry** (`{0x0, sub_0}`) sorts first
  and stops the loop immediately. → Drop the address-0 entry (and any entries
  outside `[code_base, code_base+code_size+thunk_reserve)`, which the runtime may
  reject and abort on). (`50`)
- **Entry point faults immediately reading its own stack frame.** → Initial guest
  **`r1 = stack_base`** sits on the stack's `PAGE_NOACCESS` guard page; a
  no-prologue XEX entry thunk reads its caller (loader) frame *above* `r1`. →
  Start `r1` a small 16-byte-aligned amount **below** `stack_base`. (`70`)
- **Guest `main`/entry returns instantly; game never loops; no game threads
  spawn.** → *Tempting but usually wrong* hypothesis: "the entry is a thread
  trampoline that only runs init when `r3 == -1`, so launch with
  `start_context = 0xFFFFFFFF`." We tried this on South Park and **disproved it**
  against Xenia (see next item) — the canonical launch passes `start_context = 0`
  (`r3 = 0`). Do **not** flip `r3` on a hunch. → First confirm what the entry
  actually does at that `r3` (trace it); if it returns regardless of `r3`, the
  entry is a **stub/anomaly**, not a launch bug. (`70`)
- **Recompiled entry runs but the game never starts — is it the launch or the
  binary?** → Cross-check your runtime's main-thread launch against **Xenia's
  `KernelState::LaunchModule`** (the canonical model: one `XThread` at the XEX
  entry, `start_context = 0` → guest `r3 = 0`, exit code = `r3` on return; Xenia
  uses this for *all* titles incl. XBLA — `XamLoaderLaunchTitle` is only for a
  running game launching another). If your launch matches Xenia's, the launch is
  correct and the **entry/binary is the anomaly** (e.g. a stub/mid-function entry)
  — don't keep "fixing" the launch. Verify the actual entry path with a trace
  (`r3` value, which branch). (`70`)
- **The XEX entry point lands *mid-function* (no prologue at the entry, but a later
  `addi r1,r1,N` epilogue).** → The title does **not** boot via "call the XEX entry
  on a fresh thread" — entering cold skips the `stwu` so the epilogue corrupts `r1`.
  Confirm with the `.pdata` table + a disassembler that the entry has no prologue.
  → The real `mainCRTStartup` is reached another way (kernel/loader behaviour or an
  indirect path); finding it needs a **dynamic trace or a decompiler**, not more
  launch tweaks. (`45`, `70`)
  **Dynamic signature (confirmed):** entered cold, such an entry crashes in its own
  **epilogue** — `lwz r12,-8(r1); mtlr r12; … blr` returns to a **poison/garbage**
  address (e.g. `0xBEBEBEBE` in Xenia) because the skipped prologue never saved LR.
  A trace showing the crash PC at the entry's `blr` with `r12`/`r31`=poison is the
  tell. ⚠️ **But "stock Xenia also crashes here" does NOT mean the title is
  unbootable** — that's a stock-master limitation. **Always validate against Xenia
  *canary* (far more compatible) loading the *proper STFS package*, not a loose
  extracted `default.xex`.** Real case: stock Xenia crashed at the stub `blr`, but
  **canary booted the same title to its menu from the same base xex** (no patch) once
  given the STFS package — so the entry, entered with a kernel-set-up frame + real
  content, runs the full init. Check Xenia's game-compatibility tracker for the title
  ID before concluding "research-grade." (`45`)
- **Process crashes on EXIT (nondeterministic AV / `STATUS_HEAP_CORRUPTION`), not
  during play; runs clean under a debugger.** → A **runtime shutdown/teardown**
  bug (e.g. input-listener destructor dereferencing a stale pointer), *not* guest
  memory corruption — a heap-layout Heisenbug. → Catch the throw with
  `cdb -g -G` + `sxe eh`; read the teardown stack; fix the runtime destructor. (`70`)
- **Unimplemented PPC instruction aborts a guest thread.** → The recompiler's
  `REX_UNIMPLEMENTED`/`PPC_UNIMPLEMENTED` macro **throws**; any reached
  `lmw`/`stmw`/`lq`/`stq`/`lfq*`/`stfq*`/`ba`/`bla`/exotic-VMX op kills that thread.
  → Implement the op in the recompiler's instruction dispatch/builders (load/store
  multiple & quad are mechanical loops). **Watch for asymmetric support:** a
  recompiler may implement `stmw` (used in prologues) but not its load mirror
  `lmw` (used in matching epilogues) — then a function using the multiple-word
  non-volatile-GPR save/restore pair compiles its prologue but throws in its
  epilogue. Implement both halves of every save/restore pair. *Caveat:* before
  implementing an "unimplemented" op, check it's in a real function and not
  **data-in-code** misread as instructions (clustered odd ops like `lfqu`/`lq`
  next to "unable to decode" usually mean a data region to mark, not code). (`50`)

## Dynamic analysis (Xenia as a boot-trace oracle)
- **Title load never starts in Xenia (XBLA).** → `[Content] license_mask = 0`
  blocks the license check. → Set `license_mask = 1`. (`45`)
- **Xenia hangs at startup right after the "Cache root" log.** → `discord = true`
  → `DiscordPresence::Initialize()` blocks with no Discord client / in automation.
  → Set `discord = false`. (`45`)
- **Xenia hangs at `EmulatorWindow::Create()` / won't launch a title from the
  command line in an automated (non-interactive) shell.** → It needs an interactive
  desktop to create its window. → Launch from a logged-on session (e.g. a
  `schtasks /it` task); set `log_file`/`log_level`/`log_mask` for the trace.
  *(Canary's CLI launch DID work via `schtasks /it` once the config was a fresh,
  uncorrupted one; corrupted config = silent no-launch.)* (`45`)
- **Title "doesn't boot" in your emulator test — but it should.** → You loaded a
  **loose extracted `default.xex`** (incomplete content/filesystem mount). → Load the
  **proper STFS package** (`<titleid>/000D0000/<hash>`) — Xenia mounts its full
  `StfsContainerDevice` filesystem (`\media\Assets\…`, `\UI`, …). Real case: loose xex
  only ran the entry stub; the **STFS package booted the same title to its menu**
  (~14 game threads, GPU draws). Also: **test canary, not just stock**, and check the
  **game-compatibility tracker** for the title ID before concluding it's hard. (`45`)
- **The recomp crashes but the log just ends (no error) and a live debugger hangs**
  (D3D12 window / timing). → If the runtime installs no crash handler, a guest fault
  dies silently. → Add `SetUnhandledExceptionFilter` from a static initializer in the
  app's `main` (fires only for *unhandled* faults, so it won't fight runtime SEH/MMIO
  handlers) that writes a file with the faulting address, the AV read/write target, and
  a **dbghelp `StackWalk64` of the exception `ContextRecord`**. Built RelWithDebInfo, the
  host stack **names the guest `sub_XXXXXXXX` frames** — turning a silent `0xC0000005`
  into a located call chain in one run. Decode the AV target: the runtime maps the 4GB
  guest space at a host base, so a host fault at `base + 0` (e.g. `0x100000000`) is a
  **guest null-pointer** deref. Far cheaper than running the recomp under cdb. (`80`, `95`)
- **Pinpoint a guest null/garbage pointer once the function is known**: drop a one-line
  `REXLOG_WARN` of the suspect register/field at the faulting `sub_*` (and its callers),
  rebuild, run — the value + the call site localise the root in one pass. (`80`)
- **Diagnose a live HANG (not a crash).** When the boot stops progressing (e.g. a black
  screen) instead of faulting: **attach** cdb to the running process —
  `cdb -p <pid> -c "~*k 24; qd"` (`qd` detaches and leaves it running) — to dump **all**
  thread stacks; guest frames symbolize as `module!__imp__sub_XXXXXXXX` (RelWithDebInfo).
  The "cdb hangs on the D3D12 window" caveat applies to *launching* under cdb, **not
  attaching**. Sample 2–3 times to see which threads move vs. spin. **Verify rendering** by
  screenshotting the live window *by handle* (`MainWindowHandle` → `GetWindowRect` →
  `Graphics.CopyFromScreen`; `PrintWindow` returns black for flip-model D3D swapchains).
  **Identify an unknown routine** from its `DbgPrint`/format-string arg — the `lis r,hi;
  addi r,r,lo` pair gives a guest address; read the C string there from the decrypted image
  (this is how a spin loop was identified as the Xbox-360 D3D9 GPU-hang detector). (`80`, `95`)
- **Black screen + hang: is it the GPU or game logic?** Sample the command-processor thread.
  If it's doing swaps/presents (and only *transient* `WAIT_REG_MEM` waits — confirm none is
  permanently stuck by instrumenting the `WAIT_REG_MEM` poll loop with a spin counter), the
  GPU is healthy and the game is **presenting black frames while waiting on a worker at the
  game-logic level** — debug the worker, not the GPU. (`75`, `80`)
- **An SEH "fault backtrace" finds a recovered-but-dead worker.** If the runtime *catches*
  guest faults (host `__try`/SEH), an unhandled-fault handler never fires — so a worker that
  faults-and-is-recovered leaves no crash dump, yet its work never completes and a waiter
  hangs. Log a **symbolized backtrace from inside the SEH filter** (it has the faulting
  `ContextRecord`) to locate the dead worker's fault. (`80`)

## Numeric correctness
- **Math drifts / subtle errors near zero.** → **Denormal** handling: FPU keeps,
  VMX flushes; FP state mismanaged. → Ensure per-instruction denormal mode is
  honoured; don't add stray rounding. (`50`)
- **Garbage from a kernel/struct field.** → **Endianness**: guest is big-endian;
  a shim wrote/read a field little-endian (or double-swapped). → Match guest BE
  field order; don't add manual swaps on already-swapped memory paths. (`50`/`70`)
- **Vector op produces wrong components.** → Vectors are stored **byte-reversed**;
  component order not adjusted. → Use the reversed component order the recompiler
  expects (e.g. WZYX). (`50`)

## Kernel / XAM
- **Hang/crash early in init on an unknown import.** → Unimplemented kernel/XAM
  call. → Add a logging **stub that succeeds plausibly**; implement properly only
  if it causes a visible issue. (`70`)
- **Title insists it's "not signed in" / no storage.** → `xam` user/content
  stubs too negative. → Present **one offline user + one storage device**;
  succeed content enumeration with an empty/known set. (`70`)
- **Threads deadlock or race.** → Wait/dispatcher-object semantics off. → Match
  kernel wait semantics (events/mutants/semaphores) precisely. (`70`)

## Graphics
- **Black screen but no crash.** → Command processor not translating draws, or no
  present. → Bring up in stages: cleared frame → one draw → UI → scene. (`75`)
- **Geometry exploded / wrong positions.** → **Vertex format/endian** swizzle
  (16-bit → YXWZ) or `R11G11B10` not unpacked. → Apply the re-swizzle bitmask /
  spec-constant unpack. (`60`)
- **Instanced geometry wrong/missing.** → 360 instancing is **manual and
  game-specific**. → Map the title's scheme onto modern instancing. (`60`)
- **Missing/blocky shadows, wrong cubemaps.** → Sampler LOD/filter or `cube`
  handling gaps. → Implement the needed sampler feature / cube path. (`60`)
- **Alpha-tested foliage/UI wrong.** → No fixed-function alpha test on desktop. →
  Use the alpha-test **specialization constant** (discard `< threshold`). (`60`)
- **Whole-screen color/banding off.** → EDRAM **resolve**/gamma/RT-format
  mismatch. → Fix resolve + sRGB/format handling. (`75`)

## Audio / input / saves
- **No/garbled audio.** → XMA not decoded or wrong sample-rate/channel/endian. →
  Route through the XMA decoder; fix conversion. (`75`)
- **Controller does nothing.** → XInput not bridged. → Map XInput → host
  controller; provide keyboard fallback. (`75`)
- **Saves don't persist across runs.** → Content APIs not backed by a stable host
  dir, or serialization endianness. → Back `xam` content with a fixed save dir;
  verify BE serialization. (`75`)

## Process / hygiene
- **A re-codegen wiped your edits.** → You hand-edited **generated** files. → Move
  changes into **config + `src/`** (hooks/overrides); never edit generated code.
  (`80`)
- **The recompiler emits *uncompilable* constructs you can't fix via config/`src/`**
  (undeclared-label gotos, a declared-but-undefined sentinel, out-of-range table
  entries). → Write a **reproducible post-codegen fixup script** (idempotent,
  documented, re-run after every codegen) rather than hand-editing generated
  files; provide runtime gaps as small `src/` weak stubs; keep upstream patches as
  files. (`50`)
- **A hook stopped working after a TU/opt change.** → The **address moved**. →
  Re-derive addresses; keep a hook registry and re-validate after changes. (`80`)
- **Knowledge keeps getting re-discovered.** → Findings stayed in one title. →
  **Promote general findings here**; that's the point of the KB. (`90`)
