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
- **MSVC table-based SEH = the FULL dispatch subsystem, not a config or a host-`__try`
  wrapper (the hardest part of static recomp; budget weeks).** If the title imports
  `RtlCaptureContext` + `RtlUnwind` + `__C_specific_handler` and has its own
  `RtlRestoreContext`, its `__except`/`__finally` handlers are **funclets** — *separate*
  `sub_<handler>` functions (verify: the handler PC from the `.xdata` scope table is a
  `DEFINE_REX_FUNC`, not an in-parent label). Consequences, each proven the hard way:
  (1) A host-`SEH_CATCH` in a *wrapped ancestor* can **never** reach the handler — by the
  time `RtlRestoreContext` runs, `RtlUnwind` has already unwound the `__try`-owner's frame
  off the stack, so the handler is reached by a **CONTEXT SWITCH**, not host-stack
  propagation. Making `RtlRestoreContext` *throw* just propagates to the thread root and
  kills the worker.
  (2) A codegen "catch → `goto loc_<handler>`" fails to compile (`use of undeclared label`)
  because the handler is a funclet, not a label in the parent body.
  (3) The actual fix is to emulate the dispatch: **`RtlCaptureContext`** must fill the CONTEXT
  buffer (it's often a no-op stub); **`RtlUnwind`** must walk the `.xdata` scopes, run
  `__finally`s, and set the resume target; **`RtlRestoreContext`** (`sub_XXXX`, whose tail
  `blr` jumps to `buf[<pc-offset>]`) must restore the buffer's context and **`REX_CALL_INDIRECT_FUNC(ctx.lr)`** (tail-call the handler funclet) instead of returning;
  then handle the funclet's continuation (resume after the `__try`) and the filter (pick the
  scope). All pieces are required together. Do **not** waste cycles on setjmp/longjmp config,
  host-`__try` wrapping of ancestors, or `RtlRestoreContext`→throw — they cannot work for
  table-based SEH. (`50`, `80`)
- **⚠️ BUT FIRST: is it actually table-based SEH, or a CUSTOM `setjmp`/`longjmp`?** Game
  engines often roll their **own** C++ exception/error handling (esp. for **image/asset format
  detection**: "try JPEG → on failure try TGA/PNG"), built on a `setjmp`/`longjmp` pair, NOT
  the compiler's `.xdata` SEH. South Park: LGTDP looked like SEH (it has a `RtlRestoreContext`
  = `sub_XXXX` that restores GPRs/FPRs/SP/CR/PC from a CONTEXT buffer then `blr`) but was a
  **hand-rolled setjmp/longjmp** — and that is *much* easier to fix. **How to tell:** the
  "restore" function's buffer is the SAME one a `setjmp`-like function filled in a **live
  ancestor frame** (the loader's "try"), and the call site reads as `tmp = setjmp(buf); if
  (tmp != 0) goto fail;`. **Find the TRUE pair by tracing the CALLER** (the loader), not the
  imports — the import `RtlCaptureContext` is often a **red herring** (a different buffer /
  a different mechanism); the real `setjmp` may be an indirect **EH-hook dispatcher**
  (`r0=[global]; if r0 call it; else fall through`). **Fix = `setjmp_address`/`longjmp_address`
  config on the matching pair** (recomp emits `ppc_setjmp`/`ppc_longjmp` + snapshots ctx).
  Preconditions that make it work (verify all): setjmp & longjmp use the **same guest buffer
  address**, and the **setjmp frame is still alive** when longjmp fires (nested call). If a
  prior setjmp_address attempt "regressed / aborted on no-match", it almost certainly used the
  **wrong setjmp address** (buffers didn't match) — re-derive the pair. (`50`, `80`)
- **Verify-by-running beats static reasoning for control-flow EH.** Live-instrument the
  restore site (log `buf`, `buf[pc-offset]`, the path flag) and the parser's reject path (log
  the bytes it rejects) — that's how the "it's TGA, not JPEG → legit format-detection throw"
  and "the resume buffer is never captured" facts were nailed in minutes after days of static
  guessing. A capped `static int n; if(n++<N) LOG(...)` in the generated fn + the runtime
  import is enough; generated-fn edits are wiped by regen (fine for diagnostics). (`50`)
- **`[FATAL] Unresolved call ... to 0x...` at runtime = a cross-function `b`/`goto`** the
  recompiler couldn't resolve to a function entry (a shared block / loop header in another
  fn). Harvest them all
  (`grep -rho "Unresolved call from .* to 0x[0-9A-F]*" generated/*.cpp | sed 's/.*to //' | sort -u`),
  register each as a CONFIG function so the branch becomes a tail-call ("runs that tail").
  Exclude the import-thunk band. ⚠️ Registering mid-`.pdata` targets can split functions and
  surface NEW cross-fn branches (whack-a-mole) — iterate; fix only what the boot actually hits.
- **Make the missing-function generator CUMULATIVE/idempotent.** If it derives "already
  registered" from the GENERATED code (which already contains config-added funcs), a re-run
  recomputes `(candidates − registered)` and **silently drops** all prior config entries
  (we hit 705→25). Union the existing config so the list only grows. (`50`)
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
- **`NtCreateFile` fails (`0xc000000f` NO_SUCH_FILE) on assets under a locale subdir
  (`...\en-en\foo.png`), but the file exists in the *parent* dir.** → The retail disc lays
  localized assets under a locale subdirectory (`en-en`, `Movies\en-en\`, …) and your asset
  **extraction flattened it** (dropped the locale dir). → **Runtime locale-subdir fallback:** in
  `NtCreateFile`, on a failed **pure open**, if the path has a middle component matching a locale
  tag (`xx-yy` = two ASCII letters / `-` / two letters), retry **once** with that component
  removed and serve the parent-dir file. General + title-agnostic; only fires on failure (a real
  localized file still wins), and it makes manual "hardlink the assets into an `en-en\` dir"
  setup steps unnecessary. On South Park: LGTDP this fixed the campaign level-select
  slides/diagrams (and supersedes the movie `en-en` hardlink). Tell: log shows the same path
  failing repeatedly with a `\xx-yy\` segment. (`25`, `70`)
- **A `<LevelName>.lua` (or similar per-entity script) fails to open but the level/entity still
  works.** → Many engines **probe for an optional per-level/per-entity hook script** and tolerate
  its absence (the level uses shared scripts + level *data*). If it's genuinely not in the dump
  and the content plays, it's **benign — don't chase it** (it's not a locale-path miss; the
  fallback above won't and shouldn't apply). Verify by *playing* the level. (`25`)

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
- **SOME textures corrupt (striped/garbled) — but only DYNAMIC ones, SELECTIVELY, VARYING per
  render.** ⇒ This is a **stale-data / skipped-upload race**, NOT a tile-mode/pitch/format/decode
  bug. Decode bugs are **deterministic and uniform** (corrupt every instance of a texture the same
  way, every frame); if static/menu textures are clean and only **per-frame-rewritten** textures
  (dynamic glyph caches, render-to-texture scratch) corrupt, *intermittently*, the data is fine but
  the runtime occasionally serves it from **stale guest memory because an upload was wrongly
  skipped**. The "stripes" are just stale bytes run through the (correct) untiler. → Look at the
  **shared-memory page-validity / invalidation bookkeeping** and the texture-cache re-upload path
  for a **concurrency bug**: the GPU thread's frame-end "mark pages valid" vs. the guest CPU
  thread's page-write invalidation must be **mutually atomic** (same lock). On South Park: LGTDP a
  rexglue rewrite did the valid-flag buffer swap *without* the global lock, so an invalidation
  cleared a just-retired buffer and the page stayed "valid" for one frame → the request fast-path
  skipped the re-upload → in-match dynamic text striped (menu text, a static one-time atlas, was
  always clean). Fix = take the global critical region around the swap. **Confirm cheaply** by
  forcing all uploads (e.g. disable the "mark valid after GPU write" optimization via its cvar) — if
  the corruption vanishes, it's the page-state bookkeeping, not the decoder. (`75`)
  - **FOLLOW-UP (the partial-fix trap):** on South Park: LGTDP, locking that swap **reduced but did
    not kill** the corruption — it kept recurring **intermittently, on different text each visit, and
    differed between machines** (clean on one PC, striped on another). That pattern = a **timing
    race**, and a lock that only *narrows* it means the **data in the buffer is wrong**, not just the
    access ordering. The real root cause (upstream rexglue-sdk **issue #341**) was the valid-flag
    `staging` buffer being rebuilt with an **incremental per-dirty-block copy**: `staging` is the
    buffer retired two swaps ago, so its **non-dirty blocks keep stale "valid" bits** → wrongly-valid
    pages → skipped re-upload → stale bytes. Fix = rebuild `staging` as a **FULL** snapshot every
    frame. **And** when the title re-rasterizes glyphs to the **same guest address every frame**, the
    skip optimization can *still* mark them valid-then-stale, so the reliable cure is to **default the
    optimization OFF** (force re-upload) for that title — proven clean across all screens, while the
    full-copy + read-lock keep the ON path correct for other titles. **Use the forced-upload toggle as
    the classifier first; don't trust a lock that only makes intermittent corruption rarer.** (`75`)

## Audio / input / saves
- **No/garbled audio.** → XMA not decoded or wrong sample-rate/channel/endian. →
  Route through the XMA decoder; fix conversion. (`75`)
- **Audio fidelity needs a HUMAN ear — but first prove the path objectively + capture a clip.**
  An agent can't hear, but it can settle everything *except* timbre: (1) read the conversion and
  confirm it's a correct 5.1→stereo fold (center −6 dB to L/R, surrounds summed, LFE dropped,
  normalized; BE→LE swap; channel order = XAudio2 FL,FR,FC,LFE,BL,BR); (2) add a **cvar-gated PCM
  dump** at the point you hand samples to the host audio API (e.g. just before
  `SDL_PutAudioStreamData`) → raw F32LE → `ffmpeg -f f32le -ar <hz> -ac <ch> -i dump.raw
  out.flac` (FLAC = lossless, doesn't taint the fidelity judgment); (3) check **peak (clipping)**
  and that the **non-silence duration ≈ wall-clock** (real-time). Hand the FLAC to a human on a
  real device for sign-off; never claim audio "fixed" yourself. (`75`)
- **A headless/remote box drains the audio stream FASTER than real-time (e.g. exactly 3×),
  padding your capture with silence — it's a CAPTURE-ENVIRONMENT artifact, not a port bug.**
  If the audio endpoint doesn't *pace* playback (a virtual/non-rendering sink — common over
  RustDesk/RDP or with virtual mixers), SDL's "needs more data" callback fires faster than
  real-time and your dump fills with the "queue empty → memset 0" silence branch (here: 70 %
  exact-zero frames, 3× the bytes). The **real samples are still correct and real-time** — strip
  the exact-zero frames to recover a clean clip (`max==0` per frame). On a normal output device
  SDL blocks at real-time and this doesn't happen, so **don't file it as an audio-speed bug**;
  confirm on real hardware (the human doing the ear check). Tell: dump byte-rate is an integer
  multiple of `hz·ch·4`, and stripping silence yields ≈ wall-clock duration. (`75`)
- **Controller does nothing.** → XInput not bridged. → Map XInput → host
  controller; provide keyboard fallback. (`75`)
- **Can't verify input from automation (synthetic keys don't register).** → `keybd_event`/
  `SendInput`/`PostMessage` from a background/agent process often **never reach an SDL game's
  keyboard layer** (UIPI / session-integrity isolation, or SDL ignoring synthetic input / not
  having SDL keyboard-focus) — even when `GetForegroundWindow()==game` and a focus
  transition (minimize→restore, click, AttachThreadInput) is forced. **Confirm** by logging
  the driver's key handler (e.g. mnk `OnKeyDown`): if it never fires, the keys aren't arriving.
  ⇒ **Reaching an interactive screen (title/menu) verifies boot+render+asset-load; input then
  needs a HUMAN at the keyboard/pad.** Do NOT conclude "input is broken" from failed automated
  injection — verify the *plumbing* by code (guest `XamInputGetState`/`GetKeystroke` →
  `input_system` → mnk/SDL; driver `has_focus_` default) and have a person press a key. (`75`)
- **Saves don't persist across runs.** → Content APIs not backed by a stable host
  dir, or serialization endianness. → Back `xam` content with a fixed save dir;
  verify BE serialization. (`75`)
- **The runtime save/load is provably correct, yet "continue" still resets — the GAME ignores its
  own save.** Prove the runtime end-to-end first: it WROTE the bytes (`XamUserWriteProfileSettings`/
  `XamContent*`), and it LOADED them at boot (`is_set=true` on the read). Add a temporary
  `SaveSetting` guard that refuses to overwrite a saved blob with an "emptier" one (fewer non-zero
  bytes) and confirm the **disk save survives a restart+nav**. If progress *still* resets on screen,
  the bug is **guest-side**: the title re-initialises its in-memory progress on a "new game / lobby"
  entry and never applies the loaded profile to it. **Tell-tale: the save and load touch DIFFERENT
  in-memory globals** (e.g. SAVE serializes one array, LOAD fills another) — a missing/broken
  guest-side copy. No runtime hack fixes it; it needs guest-code RE (find the apply/copy on the
  mode-entry path) + a config override / post-codegen fixup. ⚠️ And **before any of this, rule out
  TRIAL mode** (`XamContentGetLicenseMask` → `license_mask` cvar, default 0 = trial persists
  nothing; launch `--license_mask=1` for an owned copy). (`75`)
- **…and you CAN still fix it from the runtime — snapshot/restore the in-memory progress global
  directly (don't fight the disk).** Follow-up to the above: on South Park: LGTDP the "guest-side
  only" verdict was too pessimistic. The real win was finding **which in-memory global the level
  grid is actually built from**, then having the runtime persist/restore *that* — sidestepping both
  the broken guest load AND the game clobbering its own disk save. How to find it: **live
  ReadProcessMemory before/after diff across the in-session event** (e.g. winning a level): dump a
  wide region of guest memory (guest addr → host `0x100000000 + addr`, since guest space is mapped at
  host 4 GB), trigger the unlock, dump again, `diff`. The changed bytes ARE the progress state — far
  more reliable than static analysis (which gave ~6 wrong "final models" here). Gotchas that made the
  diff lie at first: (a) the progress global is **transient** (populated only around the win/save,
  reads 0 at the menu — so diff at the *right* screen, and the runtime must restore it *every frame*
  before the screen rebuilds its UI from it); (b) it's in a **per-(player)slot** block that also
  holds per-boot **heap pointers** — restore only the pointer-free int/flag sub-range, gated by a
  monotonic "progress" counter so you never reduce progress or copy a stale pointer. Implementation:
  a ~20 ms background thread captures the block to a side file when the counter grows (catches the
  transient win window; the game's own profile-write is useless because it writes DEFAULT), and the
  per-frame input hook restores it when the live block reads default. Verified end-to-end (win →
  restart → next level unlocked). **Lesson: "the game ignores its own save" does NOT mean
  unfixable-without-guest-RE — locate the in-memory state the display reads (live RPM diff) and
  snapshot/restore THAT from the runtime.** (`70`, `75`)
- **Save never fires / nothing ever persists, even though the save code is all there.**
  → **CHECK TRIAL MODE FIRST.** An XBLA title queries its license via
  **`XamContentGetLicenseMask`**, which returns the runtime's **`license_mask` cvar
  (default 0 = trial)**. In trial it shows "UNLOCK FULL GAME" / "you can keep this once you
  unlock the full game" and **deliberately persists nothing** — so you can chase the
  save-enqueue forever and never see a byte written. **Fix: launch with `--license_mask=1`**
  (the owned-game unlock; same cvar Xenia uses). On South Park: LGTDP this flipped the menu
  to full and made profile/settings writes hit disk immediately (a SETTINGS→ACCEPT rewrote
  `userdata/<titleid>/profile/User/63E83FF*` on the spot; trial wrote nothing). Tell: the
  menu's first item is "UNLOCK FULL GAME". → **Only after confirming full mode** chase the
  async machinery: console saves are often **async + request-queued state machines**, not
  synchronous writes — a per-frame "flush if dirty" pumps a queue only when the game
  **enqueues a request** / sets a dirty flag (you'll see the flush fire thousands of times
  with the flag always 0 — that's the *pump*, not the trigger). Trace to the **enqueue**;
  don't "force" the dirty gate (it pumps an empty queue). Note there are usually **two**
  paths: `XamUserWriteProfileSettings` (profile/settings, fires readily in full mode) and
  `XamContentCreateEx` (a content save-game, needs the in-game progress trigger). Leave both
  endpoints logged so a real playthrough self-reports. (`75`)
- **Boot freezes at the first frame (renders the intro, then frozen; 0 input polled;
  log stuck ~4 KB). LOOKS host-state/"reboot-only" — it is NOT.** This *presents* as a
  present/vsync deadlock that's intermittent across runs (so you're tempted to blame
  degraded driver/DWM state and reboot). On South Park: LGTDP that conclusion was **wrong**
  after ~20 ruled-out hypotheses. → **Real cause: the command processor's `WAIT_REG_MEM`
  poll loop slept a fixed `1 ms` per *unmatched* poll** (under vsync). When a fence needs
  many polls to clear, the CP crawls, falls behind the guest's frame pacing, and the guest's
  post-frame fence wait never sees catch-up → frozen at frame 1. It looks "intermittent /
  host-dependent" only because system load changes the poll count. (A second throttle:
  **per-poll logging** — thousands of log lines/sec — adds I/O that slows the CP further.)
  → **Fix: in the not-matched path, spin-yield (`SyncMemory` + `MaybeYield`) for the first
  ~8000 polls before falling back to any sleep**, and raise the stuck-detector log threshold
  far up so it isn't spamming. **No reboot, no driver issue** — verified booting straight to
  the title after the change. **Lesson: a fixed per-poll `Sleep` in a GPU busy-wait is a
  throttle; "frozen + intermittent across runs" is a pacing bug, not host state.** Diagnose
  with `cdb -p <pid> -c "~*k; qd"` and a spin counter on the poll loop; `--vsync=false` is a
  useful probe but the durable fix is the spin-yield. (`60`, `70`)
- **"Boot takes minutes" — measure before optimizing; it's often one-time COLD shader-cache
  translation, not a runtime stall.** Pin boot-to-title with **timed screenshots**, and check the
  GPU busy-wait health (e.g. a `WAIT_REG_MEM` stuck-counter = 0 → fences resolve fine). If the
  log says shaders/pipelines loaded **"from the storage"** (warm cache) and boot is now ~1 min,
  the earlier multi-minute figure was a **cold first boot** (live translation of the whole shader
  set) — inherent and cached afterward — not a bug. The residual time is usually **game-paced
  splash/intro animation** (user-skippable), which the runtime can't speed up. Don't "optimize"
  a fence loop that isn't the bottleneck. (`60`, `75`)

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
- **A *forced* trigger "verified" a feature that a *real* trigger then broke.** → When you
  cheat/mod game logic, remember the **engine (native/recompiled code) usually owns the
  outcome; the script layer (Lua/etc.) is often just presentation.** On South Park: LGTDP the
  win/lose result is decided by the engine (defense health ≤ 0 → `DefenseKilled` → its own GAME
  OVER menu); the Lua `GenericLoseLevelSequence` only plays lose music. Redefining that Lua to
  "play the win sequence" looked correct when **forced** via a direct Lua `EndGame("DefenseKilled")`
  call (showed STAGE COMPLETE) — but that was a **false positive**: forcing the script call means
  the engine never entered its real loss state. A **real** loss (zero the health so the *engine*
  detects it) still showed GAME OVER. → **Verify with the real trigger, not a synthetic one**, and
  fix at the layer that owns the decision: the working cheat **prevents the lose condition**
  (keep defense health > 0) rather than dressing up its aftermath. To make a real failure
  reproducible on demand, drive the *engine's own* state (here: `SetDefenseHealth(0)` to force a
  genuine `DefenseKilled`, and an incremental drain to stress-test the keep-alive). (`66`)
