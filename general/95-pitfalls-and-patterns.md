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
- **A hook stopped working after a TU/opt change.** → The **address moved**. →
  Re-derive addresses; keep a hook registry and re-validate after changes. (`80`)
- **Knowledge keeps getting re-discovered.** → Findings stayed in one title. →
  **Promote general findings here**; that's the point of the KB. (`90`)
