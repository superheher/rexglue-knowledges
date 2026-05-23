# Static recompilation — overview, landscape, and feasibility heuristics

## What it is

**Static recompilation ("recomp")** translates a game's compiled machine code
into portable source (here **C++**) *ahead of time*, then compiles that natively
for the host. There is no runtime CPU interpretation or JIT. The translated code
is paired with a **runtime** that implements the original platform's services
(kernel, GPU, audio, input, filesystem) at a high level — "HLE" in the emulator
sense, but linked directly into the program.

```
guest XEX (PPC)  ──recompiler──▶  generated C++  ──┐
guest shaders    ──recompiler──▶  HLSL/SPIR-V    ──┤
                                                   ├──link──▶ native PC executable
host runtime (kernel/GPU/audio/IO) ───────────────┘
```

Contrast with an **emulator**, which decodes and executes guest instructions at
runtime (interpreter or dynamic-recompiling JIT) and is title-agnostic.

## Why recomp (and why not)

**Strengths**
- **Native performance** — the guest code becomes ordinary host code the
  optimizing compiler sees; no per-instruction dispatch overhead.
- **Debuggable & moddable** — you get a normal native binary; you can hook,
  patch, profile, and add features (widescreen, higher FPS, mods).
- **Portability** — the same generated C++ + runtime can target multiple OSes
  and CPU architectures.
- **Preservation** — a self-contained native build, independent of an emulator.

**Costs / limits**
- **Per-title effort.** It is *not* push-button. Each title needs analysis
  (function boundaries, jump tables), import shims, and bring-up debugging.
- **You need the executable** and the right to use it. Bring your own dump.
- **Not general** — a recomp runs one game, not "any 360 game."
- **Hard edges**: networking, exotic GPU features, MMIO-driven hardware, and
  self-modifying / JIT guest code are the expensive parts.

Rule of thumb: **emulate to run many titles; recompile to ship one title well**
(performance, mods, a clean native port).

## The landscape / precedents

- **N64: Recompiled** (`N64Recomp`) — the modern static-recomp that popularised
  the approach; powers e.g. *Zelda: Majora's Mask "Recompiled"*. The conceptual
  parent of the 360 tools.
- **rexdex's recompiler** — the original Xbox 360 static recompiler; proved the
  PPC→C++ path for 360.
- **XenonRecomp + XenosRecomp** (hedge-dev) — CPU and shader recompilers; the
  flagship user is **Unleashed Recompiled** (*Sonic Unleashed*), a large, shipped
  result. XenonRecomp ships **no runtime** — that is the integrator's job.
- **rexglue-sdk** (ReXGlue) — an integrated SDK that does PPC→C++ codegen **and**
  ships a runtime (D3D12/Vulkan, XMA audio, SDL input, VFS, kernel/XAM), with a
  scaffolder and a PowerShell lifecycle. Heavily rooted in **Xenia**.
- **Xenia** — the Xbox 360 *emulator*; the single most valuable RE reference for
  the kernel, GPU command stream, and shader ISA. Nearly all 360 recomp work
  leans on Xenia's research.

See `99-references.md` for links.

## Feasibility heuristics (what makes a title easy or hard)

Score a candidate before committing. Easier ↔ harder:

| Factor | Easier | Harder |
|---|---|---|
| **Import surface** | only `xboxkrnl` + `xam` | also `xnet`/`xonline`, `xhv`, custom |
| **Networking** | offline / LAN | online matchmaking / dedicated servers |
| **Engine knowledge** | known engine or prior RE | bespoke, undocumented engine |
| **GPU features** | simple shaders, no exotic formats | heavy custom GPU tricks, MMIO, GPU compute |
| **Audio/video** | PCM/XMA | bespoke codecs, tight A/V sync, in-house video |
| **Code size / scope** | small (XBLA) | sprawling AAA |
| **Control flow** | clean ABI, few jump tables | many compiler-version-specific jump tables |
| **Exceptions** | no runtime C++/SEH reliance | EH used for normal control flow |
| **Existing runtime fit** | imports already covered by a runtime | many novel imports to write |

The **import surface is the strongest single predictor** of runtime effort:
list the imported libraries first (see `20-xex-format.md`), then count distinct
ordinals. A title that imports only `xboxkrnl.exe` + `xam.xex` is in the sweet
spot the existing runtimes target.

## A realistic mental model of the effort

Most of the calendar time is **bring-up debugging**, not the initial translation:

1. Translate & **compile** (days) — mechanical once configured.
2. **Boot to first frame** (weeks) — fill missing imports, fix init crashes,
   jump tables, structural recomp issues. This is the make-or-break phase.
3. **Render correctly** (weeks) — shader/format/state correctness.
4. **A/V/input/save** (weeks) — make it actually playable.
5. **Polish/perf** (weeks).

Once a title **boots and presents a frame**, the rest is iteration rather than
research — that milestone is the key early signal.

See `40-recompilation-pipeline.md` for the concrete pipeline and
`90-new-title-onboarding-playbook.md` for the step-by-step checklist.
