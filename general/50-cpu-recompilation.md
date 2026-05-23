# CPU recompilation — the hard parts and how to handle them

This is the densest part of bring-up. Most items below are configured in the
recompiler's TOML and apply to **any** 360 title; the specifics (addresses)
differ per title and come from recon. XenonRecomp's documentation is the clearest
reference for the knobs; rexglue exposes equivalents.

## How translation works (mental model)

- Each guest function becomes a C++ function taking the **PPC context** (all
  registers) and the **base pointer** (32-bit guest memory). Instructions are
  translated almost 1:1 — **not** decompiled to readable code.
- Guest memory **loads/stores byte-swap** (big-endian guest, little-endian host)
  and are **volatile** to prevent unsafe reordering.
- **Vectors** are stored byte-reversed (whole 16 bytes), so vector ops use
  adjusted component order.
- Control flow within a function is normal C++; calls dispatch to other generated
  functions; **indirect** calls go through an address→function lookup.

## Function boundary analysis

- **`.pdata`** gives start address + size for functions with unwind data — most
  functions. Use it first.
- Functions **not** in `.pdata` are found by scanning for **branch-and-link**
  (`bl`) targets and bounding them by static analysis.
- The analyzer **struggles with functions containing jump tables** (they look
  like tail calls). Fix by declaring boundaries explicitly:
  ```toml
  functions = [
    { address = 0x824E7EF0, size = 0x98 },
    { address = 0x824E7F28, size = 0x60 },
  ]
  ```

## Jump tables (switch statements)

The single most title-specific structural issue.

- Pattern: a computed target loaded into the count register then `bctr` — look
  for **`mtctr r0`** almost always followed by **`bctr`**, with preceding
  instructions computing the address.
- Detection is **compiler-version sensitive**. The bundled analyzers were tuned
  on specific titles (e.g. Unleashed); other XDK builds (older XBLA, newer SDKs)
  may use variant patterns and need analyzer tweaks or manual entries.
- Workflow: run the analyzer (`XenonAnalyse xex tables.toml` / rexglue scanners)
  → it emits a switch-table TOML → reference it from the main config so codegen
  emits **real `switch` statements**. Hand-author tables the analyzer misses.
- Symptom of a missed table: a crash/`__builtin_trap` at an indirect branch, or
  a function that "falls off" into the next one.

## Register save/restore helpers

XDK binaries call shared **non-volatile register save/restore** routines (or
inline them). The recompiler needs each routine's **start address**:

| Property | Routine | Identifying byte pattern (start) |
|---|---|---|
| `savegprlr_14` / `restgprlr_14` | GPR + LR save/restore from `r14` | `f9 c1 ff 68` / `e9 c1 ff 68` |
| `savefpr_14` / `restfpr_14` | FPR save/restore from `f14` | `d9 cc ff 70` / `c9 cc ff 70` |
| `savevmx_14` / `restvmx_14` | VMX v14..v31 | `… 7d cb 61 ce` / `… 7d cb 60 ce` |
| `savevmx_64` / `restvmx_64` | VMX128 v64.. | `… 10 0b 61 cb` / `… 10 0b 60 cb` |

Find them by the patterns above (or via the symbols if present) and put the
addresses in the config. Getting these wrong corrupts the stack/regs subtly.

## longjmp / setjmp

- The recompiler **redirects** guest `longjmp`/`setjmp` to native
  implementations (the large PPC context has room to hold host CPU state).
- Provide their addresses in the config. `setjmp` often sits **just after**
  `RtlUnwind`; `longjmp` is found via `RtlUnwind` references.
- If the title doesn't use them, omit.

## Exceptions (the structural gotcha)

- C++/SEH **exceptions are not translated**. The link register and arbitrary
  handler targets make general EH impractical in this model.
- **Exception-handling data is interleaved between functions** and is *not* valid
  code. Tell the recompiler to **skip** it via `invalid_instructions` (skip N
  bytes when a known sentinel value/word is seen), e.g. C/C++ frame handlers,
  padding, end-of-`.text` markers:
  ```toml
  invalid_instructions = [
    { data = 0x00000000, size = 4 },   # padding
    { data = 0x831B1C90, size = 8 },   # C++ frame handler
    { data = 0x00485645, size = 44 },  # end of .text
  ]
  ```
- Practical stance: most XDK titles don't use EH for *normal* control flow.
  Confirm during boot; if a title genuinely throws across translated frames,
  that's a serious (but often localized) problem to special-case.

## Indirect calls & virtual functions

- Resolved via an **address → function pointer** lookup: dereferencing a value
  derived from the original instruction address yields the recompiled function.
  Modern XenonRecomp places these lookup entries **just past the valid XEX
  memory region** in the base allocation (exported as macros in `ppc_config.h`),
  avoiding a huge sparse allocation.
- Consequence: the base allocation layout matters; don't trample the region the
  lookup relies on.
- rexglue additionally has **vtable** and **signature** scanners to recover
  indirect targets — useful when static call graphs are incomplete.

## Endianness & floating point (correctness traps)

- All guest memory access is byte-swapped; **don't** add your own swaps in shims
  unless you know the data path. Mismatched swaps are a top source of garbage
  values.
- **Denormals:** FPU preserves them, VMX flushes them. The recompiler toggles
  host denormal handling per the instruction class. Math that "almost works" but
  drifts near zero is often a denormal/rounding-mode issue.

## Optimizations (only after it runs)

Enable **after** a stable, correct boot — they assume ABI cleanliness:

```toml
skip_lr = false            # skip link-register maintenance (no exceptions)
skip_msr = false
ctr_as_local = false       # count register → local
xer_as_local = false
reserved_as_local = false
cr_as_local = false        # condition registers → locals
non_argument_as_local = false
non_volatile_as_local = false
```

`*_as_local` lets Clang keep registers in locals and **drops save/restore calls
and redundant context stores** — the biggest win (in Unleashed, ~20 MB smaller
binary and several ms/frame). Turn on incrementally and re-verify each.

## Symptoms → likely cause (quick triage)

| Symptom | Suspect |
|---|---|
| Trap/crash at an indirect branch | missing **jump table** entry |
| Function bleeds into the next / bad returns | wrong **function boundary** or save/restore address |
| Subtle math drift near zero | **denormal**/rounding (FP state) |
| Garbage from a kernel struct | **endianness** mismatch in a shim |
| Crash entering a routine that unwinds | **longjmp/setjmp** address or **EH** data not skipped |
| Works debug, breaks with opts | an `*_as_local`/`skip_lr` assumption violated |

Record every concrete instance (address + cause + fix) in the title's
`30-boot-log.md` — that journal is the most reusable artifact you produce.
