# Static binary analysis of a decrypted Xbox 360 title

Independent, read-only analysis of the title's PE image — outside the recompiler —
is the fastest way to (a) sanity-check the recompiler's view, (b) debug a boot that
doesn't progress, and (c) recover authoritative function boundaries. This is the
companion to [[20-xex-format]] and [[50-cpu-recompilation]]. Reference tools live in
`south-park-recomp/tools/` (`xex_decrypt.py`, `pe_inspect.py`, `pdata.py`,
`ppc_dis.py`, `callgraph.py`, `find_initarray.py`).

## Get the loaded image

Decrypt + basic-decompress the XEX basefile to the **loaded image** (the in-memory
layout, where **RVA == offset-from-image-base**). For basic compression: AES-CBC
(IV=0) with the session key = `AES-ECB-decrypt(file_key, retail_key)`, then
concatenate the `(data_size, zero_size)` blocks. The result starts with `MZ`.
Save it once and analyse offline. (Refs: Xenia `xex2.cc`, Free60.)

> **Endianness split (critical):** PE *headers* (DOS/NT/section/data-directory) are
> **little-endian**; all *guest content* (code, `.pdata` entries, vtables, pointer
> arrays) is **big-endian**. Read each with the right width/order or you get
> garbage (e.g. a `.pdata` table read LE looks like ASCII or `0xFFFFFFFF`).

## Disassemble (machine type 0x01F2)

Xbox 360 PEs report machine **`0x01F2` = `IMAGE_FILE_MACHINE_POWERPCBE`**.
`llvm-objdump`/most PE tools won't auto-disassemble it ("unknown arch"). Use
**capstone** `CS_ARCH_PPC + CS_MODE_BIG_ENDIAN + CS_MODE_32`, mapping
`file_offset = guest_addr - image_base`. Capstone's `disasm()` **stops at the first
byte it can't decode** (data-in-code, VMX gaps) — restart past it, or note the stop
point as a likely data region (useful signal for `[[invalid_instructions]]`).

## `.pdata` (EXCEPTION directory) = authoritative function table

The PE EXCEPTION data directory points at an array of `RUNTIME_FUNCTION` records
(8 bytes each, **big-endian**): `+0 BeginAddress` then a packed length/flags word.
This is the linker's own function list — far better than heuristic segmentation for
**function start addresses**. Use consecutive sorted `BeginAddress` values as
`[start[i], start[i+1])` spans (the packed-length field encoding is fiddly; you
rarely need it). Gotchas learned the hard way:

- **The dir RVA can be off by a page** vs a basic-decompressed image (alignment of
  the `.rdata`/`.pdata` boundary in the reconstructed image). Don't trust the dir
  RVA blindly — **auto-locate** the table by signature (first long run of ascending
  big-endian `0x82xxxxxx` values 8 bytes apart). `.text` is unaffected and
  disassembles correctly at `RVA==offset`.
- **`.pdata` merges adjacent tiny frame-sharing functions** into one record (common
  for CRT/kernel glue with frame size 0). So a `.pdata` record is *≥* one function,
  not exactly one — verify small-function boundaries by finding prologues
  (`mflr; stw r12,-8; stwu r1,-N`) rather than trusting every record start.

## Verifying the entry point

Cross-check three sources: XEX opt header `ENTRY_POINT` (0x00010100), PE
`AddressOfEntryPoint`, and the decoded bytes. Then **check the entry against the
function table**: a normal title's entry is a clean function start
(`mainCRTStartup`). If the entry lands **mid-function** (no prologue at the entry,
but an epilogue `addi r1,r1,N` downstream), the "call the XEX entry on a thread"
model is broken for that title — that is a real, diagnosable anomaly, not a
recompiler bug. (See [[95-pitfalls-and-patterns]] "launch vs binary".)

## Finding `main` when the call graph is indirect

C++-heavy titles call almost everything through **vtables / function-pointer
tables**, so a *direct* `bl` call graph is nearly useless — most functions look like
"roots," and `_initterm`/`main` have **0 direct callers**. Symptoms in tooling:
`find_initarray.py` returns giant pointer runs (those are **vtables**; a repeated
pointer is usually `__purecall`/a default thunk), and `callgraph.py roots` returns
thousands. To actually locate `main` you then need either a **dynamic trace** or a
**decompiler with indirect-xref/data-flow analysis (Ghidra/IDA)** — plain symbol
scanning won't cut it. Budget for that; don't burn days on direct-call heuristics.

## What this buys the recompiler

Even when it doesn't crack the boot, this analysis directly improves codegen: feed
`.pdata` starts as authoritative **function boundaries**, mark capstone's
decode-stop clusters as **data-in-code**, and confirm which "unimplemented op"
warnings are real code vs misread data ([[50-cpu-recompilation]]).
