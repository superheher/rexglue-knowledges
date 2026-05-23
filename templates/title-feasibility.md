# Feasibility analysis — <GAME TITLE>

**Date:** <YYYY-MM-DD> · **Title ID:** `<XXXXXXXX>` · **Platform:** Xbox 360 /
<XBLA|Disc> (<year>, dev. <studio>, pub. <publisher>)

> Copy this into `titles/<id>/00-feasibility.md` and fill in. Base the verdict on
> `general/00-static-recompilation-overview.md` heuristics and your XEX recon.

## Verdict

**<Feasible | Risky | Not worth it> — <easy | medium | hard>.** <One-paragraph
summary: what makes it easy/hard, and whether the hard risk is separable.>

| Outcome | Confidence |
|---|---|
| Extract XEX, recompile, and compile to a host binary | <High/Med/Low> |
| Boots and renders a first frame / menu | <…> |
| Playable single-player, audio, save, start→finish | <…> |
| <online / other stretch feature> | <…> |

Realistic effort: **<estimate>**.

## Why it is feasible (evidence)
1. **Import surface:** <libraries imported, e.g. xboxkrnl + xam only>.
2. **Networking:** <present/absent at import level; core-loop dependency?>.
3. **Scope/engine:** <size, known/unknown engine, middleware>.
4. **Precedent/toolchain fit:** <which toolchain, similar shipped titles>.

## Risk register
| # | Risk | Severity | Mitigation |
|---|------|----------|------------|
| R1 | <e.g. Clang not installed> | <Blocker/High/Med/Low> | <…> |
| R2 | <…> | <…> | <…> |

## Recommended scope (v1)
- **Target:** <offline single-player (+ local co-op?)>.
- **Backend:** <D3D12 on Windows / Vulkan>.
- **Out of scope:** <online / leaderboards / achievements> → stub gracefully.

## Milestones & effort
| Phase | Goal | Rough effort |
|---|---|---|
| 0 | Prereqs + build toolchain | <…> |
| 1 | Extract + XEX recon | <…> |
| 2 | First codegen → compiles | <…> |
| 3 | Boots to first frame | <…> |
| 4 | Render correct | <…> |
| 5 | Audio/input/save | <…> |
| 6 | Polish/package | <…> |

## Bottom line
<One or two sentences: go / no-go and why.>
