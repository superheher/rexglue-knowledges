# Progress report — South Park: Let's Go Tower Defense Play! recomp

Honest status of the port. **Not playable, and now assessed research-grade for this
title**; substantial, verified bring-up progress and a large reusable knowledge
base. Written/updated after the first focused agent session (2026-05-23).

> **Headline:** the pipeline works end-to-end (extract → recompile → build → boot
> the runtime → execute guest code, all verified), but the title **does not boot via
> its XEX entry**, which is a **do-nothing stub** — and this was confirmed
> **dynamically in Xenia** (stock crashes at the stub's `blr`; canary returns from it
> but the game never initializes). Even the reference emulator can't boot this title.
> The real `mainCRTStartup` is **not invoked in the normal flow** and could not be
> located by any headless static method. So *playable* needs interactive RE or a
> real-hardware boot trace — beyond autonomous static analysis. Full evidence:
> [[35-entry-forensics]]. **The recomp itself is correct** (it mirrors Xenia exactly).

## Where it got to

| Phase | State |
|---|---|
| 0 Prereqs / build rexglue | **Done** — Clang 22.1.6 + rexglue-sdk 0.8.1.4 built & installed (D3D12). |
| 1 Extract & XEX recon | **Done** — `default.xex` (8.1 MB) + 872.7 MiB asset tree extracted; recon recorded; DLC markers classified (no TU). |
| 2 Codegen & link | **Done** — 20,045 funcs / 51 TUs / 102 MiB C++ → `south_park_td.exe` (36.8 MiB) **links & runs**. |
| 3 Boot bring-up | **Blocked (research-grade)** — runtime init works and the guest entry executes, but the XEX entry is a **stub** that runs zero game/CRT code. **Dynamically confirmed**: South Park doesn't boot in Xenia (stock *or* canary) — both run only the stub. The real boot is non-standard / kernel-side; `mainCRTStartup` is not in the normal call flow and is undiscoverable headlessly. See [[35-entry-forensics]]. |
| 4–6 | Not started (gated on boot). |

## What is verified working (run, observed, logged)

The exe boots the **entire rexglue runtime**: D3D12 device (RTX 3060), XMA audio
threads, SDL input, VFS mount of the extracted assets at `game:\`, kernel module
load (221/325 xboxkrnl imports resolved, variable imports patched), shader-storage
init for title `58410931`, and it **dispatches and runs the recompiled guest entry
point**. Guest code that runs does **not** crash (proven with `cdb sxe eh`).

## What was fixed to get here (all reusable — see [[20-codegen]], [[30-boot-log]])

1. **Undeclared-label gotos** (70) from rexglue over-segmentation → reproducible
   post-codegen rewrite to tail-calls/traps (`tools/fix_recomp_labels.py`).
2. **`sub_0` link error** (address-0 sentinel had no body) → weak stub in `src/`.
3. **Function table not registering** — the zero-terminated table started with the
   `{0x0,sub_0}` entry → drop it (range filter in the same tool).
4. **Runtime aborted on 41 out-of-range table entries** → range filter drops them.
5. **Entry faulted on the stack guard page** — initial `r1 = stack_base` → reserve
   headroom below it (runtime patch `0001`).
6. **`lmw` epilogues threw `REX_UNIMPLEMENTED`** — codegen had `stmw` but not its
   load mirror → implemented `build_lmw` (runtime patch `0003`; `lmw` 119 → 0).

*(A 7th change — launching with `r3 = 0xFFFFFFFF` on the theory the entry runs init
only when `r3 == -1`, patch `0002` — was later **disproved against Xenia and
reverted**; the entry returns regardless of `r3`. See [[30-boot-log]].)*

Plus the exit crash was correctly attributed to a **rexglue input-teardown bug**
(not guest corruption).

## Real effort / approach

One session. Heavy use of: a hand-written STFS extractor + XEX recon (Python),
`cdb` (Windows Kits) for symbolized crash triage and `sxe eh`, source reading of
the rexglue runtime, and a reproducible post-codegen fixup. The recompiler
(rexglue 0.8, early-development) needed several small fixes but did the heavy
lifting; none of the blockers were research-grade — they were bring-up plumbing.

## Top reusable lessons (promoted to general/)

- STFS block→offset math + contiguous fast path (`25`); XEX import counts readable
  pre-decompression (`20`).
- The recompiler-output gotchas now in `95`: undeclared-label gotos, the
  zero-terminated func-table + address-0 entry, out-of-range table entries, the
  `r1`-on-guard-page entry fault, the **launch-vs-binary** discriminator
  (cross-check launch against Xenia before flipping `r3` — we did, and reverted a
  wrong `r3==-1` patch), the shutdown-teardown Heisenbug, and `REX_UNIMPLEMENTED`
  throwing.
- Methodology: reproducible post-codegen fixups beat hand-edits; keep upstream
  patches as files; `cdb -g -G -cf` + `sxe eh` is the fastest crash/throw triage.

## Honest remaining path (now: research-grade, not just multi-week)

The next milestone (first rendered frame) requires getting the title's **real
init/main to run**, and this session **escalated the difficulty** from "multi-week
bring-up" to **research-grade for this title**, with dynamic proof:

1. The XEX entry (`0x824499A0`) is a **stub** (confirmed by XEX/PE headers,
   independent decrypt, `.pdata`, Ghidra). Entered the standard way it runs no
   game/CRT code.
2. **Xenia can't boot it either** (stock crashes at the stub's `blr`; canary returns
   from the stub but the game never initializes) — so this is *not* a recomp defect
   and *not* solvable by the standard launch model. [[35-entry-forensics]]
3. The real `mainCRTStartup` is **not invoked in the normal flow** and could not be
   located by 5+ headless static methods (the binary is extremely C++/vtable/
   singleton/VMX128-dense). [[35-entry-forensics]] table.

**What would actually unblock it (pick one):**
- **Interactive decompiler (Ghidra/IDA GUI)** — a human-driven session navigating
  from the `.CRT` section / init array / `exit`-terminate imports to find the real
  CRT entry, then start the recomp there via the prepared `REX_ENTRY_OVERRIDE` hook.
  (Maintainer has Ghidra + RE expertise.)
- **Real-hardware (or JRPCS3-style) boot trace** — capture how the kernel actually
  drives this title's init past the stub.
- **Pivot** to a title that *does* boot in Xenia (verifiable up front) for the
  *playable* goal; South Park remains a strong KB case study.

After boot reaches a frame, Phases 4–6 (rendering, audio, input, save) are the
"normal" multi-week iteration the original estimate covered. This session cleared
Phases 0–2, all the Phase-3 *plumbing*, and **definitively characterised the
Phase-3 blocker** (the largest reusable outcome).

## Top reusable lessons from this session (promoted to general/)

- **Use Xenia as a boot-trace oracle** (stock *and* canary) to validate a recomp's
  launch and to tell "hard title" from "recomp bug": if the reference emulator dies
  at the same instruction, the recomp is correct. Setup gotchas (license_mask,
  discord hang, GUI-only launch in some builds) in `general/45`+`95`.
- **`.pdata` is the authoritative function table** (big-endian; auto-locate; merges
  tiny funcs); machine type `0x01F2` needs capstone BE-PPC w/ `skipdata`. `general/45`.
- **A mid-function/stub XEX entry is a real, diagnosable anomaly** with a clean
  dynamic signature (epilogue `blr` to a poison LR). `general/95`.
- **CRT-entry signature hunts (`_initterm`, init-array, security-cookie) drown in
  C++ noise** in heavy titles — don't expect headless heuristics to pin
  `mainCRTStartup`; use interactive RE or a trace. `general/45`.
