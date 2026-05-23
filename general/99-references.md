# References

Curated, with one-line "why it matters". Verify versions/links before relying on
specifics; treat upstream docs as canonical over this KB where they disagree.

## Recompilation tools
- **XenonRecomp** — https://github.com/hedge-dev/XenonRecomp — PPC→C++ recompiler;
  its README is the clearest spec of the per-title config knobs.
- **XenosRecomp** — https://github.com/hedge-dev/XenosRecomp — Xenos shader →
  HLSL/DXIL/SPIR-V; README documents the shader gotchas.
- **rexglue-sdk** — https://github.com/rexglue/rexglue-sdk — integrated codegen +
  runtime; see its **wiki** (Getting Started, Codegen Pipeline, ReXApp, Kernel
  State, Memory, VFS, Mid-ASM Hooks, Function Overrides).
- **rexdex recompiler** — https://github.com/rexdex/recompiler — the original 360
  static recompiler; historical/reference.
- **N64: Recompiled** — https://github.com/N64Recomp/N64Recomp — the conceptual
  parent; the static-recomp approach generalized.

## Reference ports (read their source)
- **Unleashed Recompiled** — https://github.com/hedge-dev/UnleashedRecomp — the
  flagship XenonRecomp+XenosRecomp result; example config (`SWA.toml`),
  switch-table TOML, and runtime patterns to copy.

### ReXGlue reference ports (same toolchain as rexglue-sdk — invaluable)
Read their `*_config.toml` + `config/*.toml` + `src/` for the working bring-up
pattern (the bare `rexglue init` scaffold is *not* enough to boot a game):
- **Viva Piñata: TiP** — https://github.com/SolarCookies/TiP-Recomp — **playable**;
  config split into `retip_crt.toml` ([rexcrt] heap/CRT→native), `retip_midasm.toml`,
  `retip_hooked.toml`, `retip_ctx_ctr.toml`; `setjmp_address`/`longjmp_address` set;
  app overrides path wizard + overlays. **Commits its generated code** — its
  `xstart` is a model of a normal recompiled CRT entry (prologue +
  `bl __savegprlr_*` + `stwu r1,-N` + init calls + `main`).
- **Banjo-Kazooie: Nuts & Bolts (reNut)** — https://github.com/masterspike52/reNut
  — boots; `renut_crt.toml` [rexcrt] (RtlAllocateHeap/memcpy/str*/…), `[functions]`
  boundary list, `[[midasm_hook]]` gameplay hooks; clean `renut_app.h`.
- **Ace Combat 6** — https://github.com/rapidsamphire/AC6Recomp (+ D3D12 fork
  https://github.com/sal063/AC6_recomp) — D3D/render hooks (`d3d_hooks`,
  `render_hooks`).
- **Blue Dragon (reblue)** — https://github.com/zolaware/reblue and the official
  test repo https://github.com/rexglue/reblue — title screen boots; large reversed
  engine under `src/bdengine/`.

**Bring-up pattern to copy from these:** map the guest CRT via `[rexcrt]` (heap is
all-or-nothing), set `setjmp`/`longjmp` addresses, add `[functions]` for
analyzer-missed boundaries and `[[midasm_hook]]`/function overrides for engine
fixes, split config across `config/*.toml` via `includes`. A **normal title's
entry point is a full CRT startup** (`__savegprlr_*` + large frame + `_initterm`
init + `main`) — if your recompiled `xstart` is a tiny mid-function stub, the
entry point or its analysis is the problem (see this KB's South Park case study).

## Emulator / RE sources (the research backbone)
- **Xenia** — https://github.com/xenia-project/xenia — the single most valuable
  reference for kernel imports, the GPU command stream, and the shader ISA.
- **Xenia Canary** — https://github.com/xenia-canary/xenia-canary — active fork;
  XEX patching code among other things.
- **Free60 Project** — https://github.com/Free60Project (and the Free60 wiki) —
  community documentation of XEX, STFS, kernel, hardware.

## Formats & hardware
- **XEX2 / STFS / GDFX** — documented across Free60 and Xenia source; see
  `20-xex-format.md` and `25-containers-and-extraction.md` here for the
  recomp-relevant summary.
- **PowerPC / VMX128** — vendor PPC manuals + AltiVec; the rexglue-sdk repo also
  carries `docs/ppc/` (AltiVec & core instruction PDFs, `vmx128.txt`) as a handy
  in-tree reference.

## Extraction utilities
- **extract-xiso** — https://github.com/XboxDev/extract-xiso — ISO/GDFX extract &
  rebuild.
- **wxPirs / Velocity / Horizon** — community STFS package extractors/editors
  (find via the Xbox 360 homebrew community / their project pages).
- **QuickBMS** — scriptable extractor; community STFS scripts exist.

## Techniques / articles
- **DXIL linking for specialization constants** —
  https://therealmjp.github.io/posts/dxil-linking/ — the trick XenosRecomp uses to
  emulate spec constants under DXIL.

## How to cite in this KB
When you rely on a specific behaviour, link the exact source (tool README
section, Xenia file, or Free60 page) next to the claim, especially for empirical
specifics that this KB asserts. Keep `general/` citable so it can be published
standalone.
