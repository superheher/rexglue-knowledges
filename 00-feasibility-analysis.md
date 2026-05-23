# Feasibility analysis — recompiling *South Park: Let's Go Tower Defense Play!*

**Date:** 2026-05-23 · **Title ID:** `58410931` · **Platform:** Xbox 360 / XBLA
(2009, dev. Doublesix, pub. Microsoft Game Studios)

## Verdict

**Feasible — medium difficulty.** This is a *good* recompilation candidate: a
small 2009 XBLA title with the minimal Xbox 360 import surface, attacked with a
toolchain (rexglue-sdk; XenonRecomp/XenosRecomp) that has already shipped a far
larger game (*Sonic Unleashed*). The hard, open-ended risk (online multiplayer)
is separable from the playable core.

| Outcome | Confidence |
|---|---|
| Extract XEX, recompile, and **compile** to a host binary | **High** |
| **Boots and renders** a first frame / menu | **Medium-High** |
| **Playable single-player** (+ local co-op), audio, save, start→finish | **Medium** |
| Online co-op working | **Low** (out of scope for v1; deferrable/stub) |

Realistic effort for a focused developer (or a well-driven agent loop):
**~2–4 months part-time** to playable single-player; "boots & renders" is
reachable much sooner. See the milestone table at the end.

---

## Why it is feasible (evidence)

1. **Minimal import surface.** A read-only scan of the package shows the title
   imports only **`xboxkrnl.exe`** (kernel) and **`xam.xex`** (app manager).
   These two libraries are exactly what the rexglue / Unleashed runtimes
   implement. There is **no `xnet`/`xonline`/`xbdm` import library** present —
   the most expensive surface to emulate (networking, debug) is small or absent.
   See [[20-dump-analysis]] for the raw findings.

2. **No networking dependency at the binary level.** XBLA co-op was online, but
   without `xnet` in the import table the networked code path is narrow. The
   **offline single-player + local co-op** core can be built without touching
   netcode; online can be stubbed to "unavailable".

3. **Small, simple game.** A 2.5D tower-defense title has *modest* shader
   variety, *modest* CPU/AI complexity, and a *small* code section relative to a
   3D action game. Fewer unique shaders and fewer exotic GPU features means less
   of XenosRecomp's game-specific shader work bites.

4. **Strong precedent.** *Unleashed Recompiled* (Sonic Unleashed) — a large,
   shader-heavy 3D engine — was shipped on this exact toolchain. A small XBLA
   title is comfortably inside the envelope the tools have already cleared.

5. **A purpose-built, integrated SDK exists.** `rexglue-sdk` provides, in one
   place: a phased PPC→C++ **codegen**, a **D3D12 *and* Vulkan** renderer with
   **its own Xenos shader translation**, **XMA audio** (FFmpeg/xenia fork) +
   SDL audio, SDL input, a virtual filesystem, kernel/XAM objects, a project
   **scaffolder** (`rexglue init`), and a **PSReX** PowerShell lifecycle that
   fits this Windows host. See [[10-toolchain]].

6. **Standard, well-understood container.** The dump is a LIVE-signed STFS
   package with `default.xex` present (XEX2). Extraction is a solved problem.

---

## Why it is non-trivial (risk register)

| # | Risk | Severity | Mitigation |
|---|------|----------|------------|
| R1 | **Clang toolchain absent on host.** rexglue & recomp output require Clang 18–20+; only MSVC-less CMake/Ninja are present today. | Blocker (easy) | Install LLVM/Clang 20+ (or VS2022 "C++ Clang tools"). Phase 0. |
| R2 | **rexglue-sdk is "early development."** Public API churn, rough edges. | Medium | Pin a submodule commit; keep XenonRecomp/XenosRecomp as a cross-check/fallback path. |
| R3 | **Jump-table & function-boundary detection is compiler-version-sensitive.** XenonAnalyse is tuned to Unleashed's XDK; a 2009 XBLA XDK build may differ. | Medium | Iterate the analyzer; supply manual `functions`/`switch_table` entries in TOML; use rexglue's vtable/sig scanners. |
| R4 | **C++/SEH exceptions are not translated** by XenonRecomp (rexglue codegen is similar). If the title relies on EH for control flow, gaps appear. | Medium | Skip EH data as `invalid_instructions`; most XDK titles don't use EH for normal flow. Confirm during boot bring-up. |
| R5 | **Per-game shader semantics** (vertex fetch, declarations, instancing, R11G11B10, alpha-test) are manual in XenosRecomp. | Medium | Lean on rexglue's runtime shader path first; fix visual bugs case-by-case. Small game = small surface. |
| R6 | **Custom Doublesix engine** with little public RE. Game-specific systems (save, timing, asset I/O) need first-party reversing for hooks. | Medium | Mid-asm hooks + function overrides (both toolchains support this); reverse only what blocks progress. |
| R7 | **XMA audio / any Bink-style video** cutscenes. | Low-Med | rexglue ships XMA decode + FFmpeg; wire to SDL audio. |
| R8 | **Online co-op** (matchmaking/session). | High | **Out of scope for v1.** Stub `xam`/session APIs to "offline"; keep local play. |
| R9 | **Title update / DLC** (two small marketplace packages in the dump). | Low | Identify whether a TU (`.xexp`) exists; if so, feed it to the recompiler's patch path. See [[20-dump-analysis]]. |

None of R1–R7 are novel — each has an established play in prior recomp projects.
R8 is the only genuinely hard item and it is *separable* from the playable core.

---

## Recommended scope for "playable on this host"

- **Target:** offline **single-player** campaign + **local co-op** if cheap.
- **Backend:** **D3D12** (native Windows 11 host); Vulkan as fallback.
- **Must-have:** boot → menus → in-game render → input → audio → save/continue.
- **Out of scope (v1):** online co-op, leaderboards, achievements sync, avatar
  awards. Stub to graceful "offline".

---

## Milestones & rough effort

| Phase | Goal | Rough effort |
|---|---|---|
| 0 | Prereqs: Clang 20+, CMake 3.25+, Ninja; build rexglue / recompilers | hours |
| 1 | Extract `default.xex` (+ TU if any) from STFS; XEX recon (sections, imports, entry, base) | 0.5–1 day |
| 2 | `rexglue init` → config TOML → first **codegen** → **compiles** (may not run) | a few days |
| 3 | **Boots** to first frame / title; survive init (kernel stubs, jump tables, longjmp/setjmp, EH skips) | 1–3 weeks |
| 4 | **In-game rendering** correct (shader translation, vertex formats, render state) | 2–6 weeks |
| 5 | **Audio + input + save + timing**; menu→match→win/lose loop solid | 2–4 weeks |
| 6 | Polish, perf (enable codegen optimizations), packaging | 1–2 weeks |

"Boots & renders something" (end of Phase 3) is the key early signal that the
remainder is mostly iteration rather than research.

---

## Bottom line

If you only get one sentence: *a tiny-import-surface 2009 XBLA tower-defense
game is one of the friendlier things you can point this toolchain at, the core
risk (netcode) is optional, and the main cost is iterative bring-up — not a
research gamble.* Proceed.

See [[10-toolchain]] for the toolchain decision and [[20-dump-analysis]] for the
measurements behind this verdict.
