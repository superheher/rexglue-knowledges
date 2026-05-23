# Entry-point forensics — why the boot stalls (definitive)

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

## Dynamic trace via Xenia — setup (in progress)

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
