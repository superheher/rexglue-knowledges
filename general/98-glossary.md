# Glossary — Xbox 360 / static-recompilation terms

- **Static recompilation ("recomp")** — translating a game's machine code to
  portable C++ *ahead of time*, then compiling it natively. Unlike an emulator,
  there is no runtime CPU interpretation/JIT. Precedents: N64 Recomp, rexdex's
  recompiler, XenonRecomp, *Unleashed Recompiled*.

- **XEX / XEX2** — the Xbox 360 executable format (a PE wrapped with an XEX2
  header carrying import tables, security info, and a possibly LZX-compressed,
  AES-encrypted PE image). The game EXE is conventionally `default.xex`.

- **XEXP (`.xexp`)** — an XEX **patch** delivered by a Title Update; applied to
  the base XEX to produce the patched executable that is actually run/recompiled.

- **STFS** — *Secure Transacted File System*, the container used for XBLA games,
  saves, DLC, etc. Signing variants by magic at offset 0: **`CON `** (console),
  **`LIVE`** (Xbox Live), **`PIRS`** (Microsoft-signed offline). Data region
  begins at `0xC000`; 4 KiB blocks; hash blocks interleave every `0xAA` blocks.

- **Title ID** — 32-bit game identifier (here `58410931`); also the top content
  directory name.

- **Content type** — what a package holds, e.g. `0x00000002` Marketplace
  Content, `0x000D0000` (used here for the main package).

- **Xenon** — the Xbox 360 **CPU**: 3-core, in-order PowerPC, big-endian, with
  the **VMX128** SIMD extension (128 vector registers).

- **Xenos** — the Xbox 360 **GPU** (ATI/AMD); shaders are microcode in a
  reflection-bearing container, translated to HLSL→DXIL/SPIR-V.

- **VMX128** — Xenon's 128-register vector SIMD ISA; recompiled via x86
  intrinsics (or SIMDe on ARM64). Endianness handled by reversing whole vectors.

- **xboxkrnl.exe** — the Xbox 360 kernel import library (memory, threads, sync,
  I/O, timing). First thing a runtime implements.

- **xam.xex** — Xbox Application Manager import library (UI/blades, content &
  storage, profiles, notifications, marketplace). Largely stubbable offline.

- **XMA** — Xbox 360 audio codec (decoded on hardware via MMIO on console).
  Provided here by rexglue's FFmpeg (xenia fork) + XMA path, fed to SDL audio.

- **Jump table / switch table** — compiler-generated indirect-branch tables
  (`mtctr`/`bctr`); must be discovered statically (XenonAnalyse) or declared in
  TOML so codegen emits real `switch` statements.

- **Function boundary analysis** — determining where functions start/end (from
  `.pdata` or by scanning for branch-link); ambiguous around jump tables, so the
  TOML allows explicit `functions = [{ address, size }]`.

- **Register save/restore functions** (`__savegprlr_14`, `__restvmx_64`, …) —
  XDK helper routines for non-volatile registers; their addresses must be given
  to the recompiler.

- **Mid-asm hook** — a host C++ callback injected at a specific guest
  instruction address (without overwriting it), for patching behaviour.

- **Function override** — replacing a whole recompiled guest function with a
  host implementation while keeping access to the original (weak-alias trick).

- **PPC context** — the struct of all guest CPU registers passed to every
  recompiled function; the translation operates little-endian and byte-swaps on
  guest memory load/store.

- **rexglue / ReXGlue** — integrated Xbox 360 recomp **SDK + runtime** used here
  as the primary toolchain. See [[30-toolchains]].

- **PSReX** — rexglue's PowerShell module exposing `rex-configure` / `rex-build`
  / `rex-test` / `rex-format` / `rex-lint` / `Invoke-ReXSetup`.
