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

## CPU structural
- **Trap/`__builtin_trap`/crash at an indirect branch.** → Missing **jump table**
  entry. → Add it via `XenonAnalyse`/scanner output or hand-author in the switch
  table TOML. (`50`)
- **A function bleeds into the next; bad returns/stack.** → Wrong **function
  boundary** or wrong **register save/restore** address. → Declare boundaries in
  `functions`; fix save/restore addresses by byte pattern. (`50`)
- **Crash entering a routine that unwinds.** → `longjmp`/`setjmp` not redirected,
  or **EH data** parsed as code. → Set their addresses; add `invalid_instructions`
  skips. (`50`)
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
- **Guest `main` returns instantly; game never loops; no game threads spawn.** →
  XDK entry thunks double as the thread trampoline and run process init only when
  **`r3 == -1`**; the launcher passed `start_context = 0`. → Launch the main
  thread with `start_context = 0xFFFFFFFF`. (`70`)
- **Process crashes on EXIT (nondeterministic AV / `STATUS_HEAP_CORRUPTION`), not
  during play; runs clean under a debugger.** → A **runtime shutdown/teardown**
  bug (e.g. input-listener destructor dereferencing a stale pointer), *not* guest
  memory corruption — a heap-layout Heisenbug. → Catch the throw with
  `cdb -g -G` + `sxe eh`; read the teardown stack; fix the runtime destructor. (`70`)
- **Unimplemented PPC instruction aborts a guest thread.** → The recompiler's
  `REX_UNIMPLEMENTED`/`PPC_UNIMPLEMENTED` macro **throws**; any reached
  `lmw`/`stmw`/`lq`/`stq`/`lfq*`/`stfq*`/`ba`/`bla`/exotic-VMX op kills that thread.
  → Implement the op in the recompiler's instruction dispatch/builders (load/store
  multiple & quad are mechanical loops). (`50`)

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
