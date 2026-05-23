# Progress report — South Park: Let's Go Tower Defense Play! recomp

Honest status of the port. **Not yet playable, but the boot IS reproducible** (Xenia
canary boots it to the menu from the same base xex); substantial verified bring-up +
a large reusable KB. Written/updated 2026-05-23.

> ## ⚠️ CORRECTION — the title boots in Xenia (earlier "research-grade" was wrong)
> A late-session web/Xenia check overturned the mid-session "doesn't boot in Xenia /
> research-grade" verdict. **Xenia canary fully boots South Park** from the **proper
> STFS package** (base `default.xex`, **no patch**): ~14 game threads, content load,
> audio init, GPU draws — Xenia compat #1156 = **"state-menus"**. My earlier "doesn't
> boot" runs used the **loose extracted xex** (setup error). So the boot is
> **achievable**; the recomp's early-return at the entry is a **fixable
> runtime/content-mount discrepancy**, now diagnosable against the working canary
> trace — *not* a research-grade wall. The "research-grade" framing below is
> superseded; treat it as "non-trivial bring-up with a working emulator reference."

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
| 3 Boot bring-up | **In progress (iterating)** — the recomp now **boots through the guest CRT** into game subsystem init. (The earlier "research-grade / doesn't boot" verdict was a **setup error**: the recomp ran on a *corrupt* loose `default.xex` from a bug in my own `tools/stfs_extract.py`; canary **does** boot the title to menus (compat #1156) from the STFS package. Both corrected — see [[35-entry-forensics]].) Boot now runs xstart → CRT → subsystem/handler init → party/session writer init → GPU `SetInterruptCallback`, fixing bugs as they surface (content corruption, boot continuation, null EH hook, analyzer-missed functions). See [[35-entry-forensics]]. |
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

## Honest remaining path (tractable iterative bring-up — multi-week+)

> **Retraction (2026-05-23):** an earlier version of this section called the boot
> "research-grade / doesn't boot anywhere." That was a **setup error**, now
> corrected and superseded. Two root causes had masked the real boot: (1) the
> recomp ran on a **corrupt** loose `default.xex` (a bug in my own
> `tools/stfs_extract.py` STFS block math — `.text` survived so the entry
> disassembled, but `.data`/assets were garbage), and (2) the boot **continuation**
> wasn't wired (the XapiThreadStartup trampoline's `blr` needs the dispatcher to
> follow it). With correct content + the continuation fix, the recomp **boots
> through the guest CRT**, and **canary boots the title to menus** (compat #1156)
> from the STFS package. The XEX entry `0x824499A0` is the normal XapiThreadStartup
> trampoline, not a dead stub. See [[35-entry-forensics]].

The boot is now an **iterative bring-up**, not a research problem. Each fault is a
concrete, fixable bug; the loop is: run → read the FATAL/crash → fix → rebuild →
the boot advances. Verified progression this session: `xstart` → CRT → subsystem/
handler init → party/session **writer init** (was a hard crash; fixed) → GPU
`SetInterruptCallback` → deeper subsystem registration → next missing function.

**Remaining work to a first frame:** keep clearing the current fault class —
**"Call to invalid or unregistered function at 0x..."** (indirect-call targets
rexglue's analyzer misses) — via the `[functions]` config (see [[20-codegen]] and
the catalog entry in `general/95`). A scan-generated batch (`tools/gen_missing_funcs.py`)
registers the static-vtable class in one regen; the computed-jump class is added as
each surfaces. After the boot stops faulting on missing functions it will reach the
render/update loop, and Phases 4–6 (rendering correctness, audio XMA→SDL, input,
save) are the "normal" multi-week iteration the original estimate covered. Realistic
total to *playable*: multi-week to a few months, dominated by Phase 4–6 correctness,
not by any single blocker.

## Top reusable lessons from this session (promoted to general/)

- **Use Xenia/canary as the ground-truth oracle**, but **verify the inputs first** —
  the single biggest time sink here was debugging a recomp running on *corrupt
  extracted content*; a `.text`-only-correct dump boots far enough to look real.
  Cross-check `.data` against the reference emulator early. `general/95`.
- **`.pdata` is the authoritative function table** (big-endian; auto-locate; merges
  tiny funcs); machine type `0x01F2` needs capstone BE-PPC w/ `skipdata`. `general/45`.
- **The analyzer misses indirect-call targets** (vtable methods + computed-jump/
  adjustor-thunk targets); runtime FATALs them as "unregistered function." Fix via
  the recompiler's explicit-`functions` config + regen, not by hand-emitting each;
  **never** register an import-thunk address (→ undefined `sub_`). `general/95`, `general/50`.
- **Instrument the recomp itself** (a one-line `REXLOG_WARN` of the suspect pointer/
  gate) to turn a silent null-deref into a root cause in one run; the runtime has no
  exception handler, so a hard fault just ends the log. `general/95`.
- **A null runtime hook can silently skip setjmp-style try bodies** — if a dispatcher
  calls `[global_fn_ptr]` only when set and the pointer is image-zero/uninstalled, the
  faithful fallthrough returns a stale value the caller reads as "exception taken,"
  skipping critical init. Default such a dispatcher to the "no-jump" value. `general/80`.
