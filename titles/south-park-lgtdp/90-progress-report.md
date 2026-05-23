# Progress report — South Park: Let's Go Tower Defense Play! recomp

Honest status of the port. **Not yet playable**; substantial, verified bring-up
progress and a large reusable knowledge base. Written after the first focused
agent session (2026-05-23).

## Where it got to

| Phase | State |
|---|---|
| 0 Prereqs / build rexglue | **Done** — Clang 22.1.6 + rexglue-sdk 0.8.1.4 built & installed (D3D12). |
| 1 Extract & XEX recon | **Done** — `default.xex` (8.1 MB) + 872.7 MiB asset tree extracted; recon recorded; DLC markers classified (no TU). |
| 2 Codegen & link | **Done** — 20,045 funcs / 51 TUs / 102 MiB C++ → `south_park_td.exe` (36.8 MiB) **links & runs**. |
| 3 Boot bring-up | **Partial** — full runtime init works; guest entry executes **without faulting**; but the entry does not reach the game's real startup (zero kernel calls) → **no rendered frame yet**. |
| 4–6 | Not started. |

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
6. **Entry skipped init** — main thread launched with `r3 = 0`; the XDK entry runs
   init only when `r3 == -1` → launch with `0xFFFFFFFF` (runtime patch `0002`).

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
  `r1`-on-guard-page entry fault, the `r3==-1` entry sentinel, the
  shutdown-teardown Heisenbug, and `REX_UNIMPLEMENTED` throwing.
- Methodology: reproducible post-codegen fixups beat hand-edits; keep upstream
  patches as files; `cdb -g -G -cf` + `sxe eh` is the fastest crash/throw triage.

## Honest remaining path (the multi-week core)

The next milestone (first rendered frame) requires getting the title's **real
init/main to run** — the executed entry is a trivial thunk that returns. Concrete
next steps are in [[30-boot-log]] (verify/disassemble the entry; TLS callbacks /
static initializers; `XapiThreadStartup` trampoline; fix the partially
mis-recompiled early-code region `0x82131xxx` — implement `lmw`/`lq`/`stq`/`lfq*`
etc. and mark data-in-code). After boot reaches a frame, Phases 4–6 (rendering,
audio, input, save) are iteration. Per the feasibility estimate this is ~2–4
months part-time; this session cleared Phases 0–2 and most of the Phase-3
plumbing.
