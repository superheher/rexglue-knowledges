# New-title onboarding playbook

A reusable, opinionated, step-by-step procedure for taking **any** Xbox 360 title
from a dump to a playable native build. Copy the checklists from `templates/`
into the new title's case-study folder and tick them off. Cross-references point
to the deep-dive docs.

> The meta-rule: **reach a booting frame as fast as possible, then iterate.**
> Stub generously; implement properly only where a stub causes a visible problem.

---

## Stage A — Project setup (once)

- [ ] Create a **super-repo** for the title (or reuse your standard layout).
- [ ] Add toolchain **submodules**: `rexglue-sdk` (primary) + `XenonRecomp` +
      `XenosRecomp` (reference/fallback) under `third_party/`.
- [ ] Add **this knowledge base** as a submodule (so findings are shared/reused).
- [ ] Create the **port repo** (`config/`, `tools/`, `src/`, `docs/`, `generated/`).
- [ ] `.gitignore` the dump, `private/`, `extracted/`, `generated/`, `*.xex*`.
- [ ] Decide commit/author conventions; English-only content.

## Stage B — Acquire, identify, extract (Phase 1)

- [ ] Identify the **container** (magic/structure): STFS `CON`/`LIVE`/`PIRS`,
      GOD, or ISO/GDFX (`25-containers-and-extraction.md`).
- [ ] Note the **Title ID** and content types.
- [ ] **Extract** `default.xex` → `private/`; verify first 4 bytes `XEX2`.
- [ ] **Extract the asset tree** → a content root for the VFS.
- [ ] Find & classify **Title Updates / DLC**; keep any `.xexp`.

## Stage C — Recon & go/no-go (Phase 1)

- [ ] Run **XEX recon** and fill the checklist in `20-xex-format.md`
      (base, entry, imports/ordinals, `.pdata`, compression/encryption, TU).
- [ ] **Score feasibility** with the heuristics in
      `00-static-recompilation-overview.md` (import surface, networking, engine,
      GPU features, scope, exceptions).
- [ ] **Decide scope**: what "playable v1" means (usually offline single-player;
      defer online). Write the title's `00-feasibility.md`.
- [ ] **Go/no-go gate.** If the import surface is small (≈ `xboxkrnl` + `xam`) and
      no online dependency for the core loop → **go**. If it needs heavy
      networking/exotic GPU for the core loop → reconsider scope or title.

## Stage D — Toolchain (Phase 0, can overlap)

- [ ] Install **Clang 20+**, CMake 3.25+, Ninja, `pwsh` 7 (`30-toolchains.md`).
- [ ] `git submodule update --init --recursive`.
- [ ] Build/install **rexglue-sdk**; confirm the CLI runs.

## Stage E — Scaffold & first codegen (Phase 2)

- [ ] `rexglue init --app_name <app> --app_root <port>`.
- [ ] Point config `file_path` at `private/default.xex` (+ patch path if TU).
- [ ] Add structural config (`50-cpu-recompilation.md`):
  - [ ] register **save/restore** addresses (byte-pattern search)
  - [ ] `longjmp`/`setjmp` addresses (if used)
  - [ ] **switch tables** from `XenonAnalyse`/scanners; hand-author misses
  - [ ] explicit `functions` for jump-table-laden routines
  - [ ] `invalid_instructions` to skip EH/padding
- [ ] **Codegen** with `--force`; iterate until it completes.
- [ ] **Compile**. Gate: *the app links into an executable.*

## Stage F — Boot bring-up (Phase 3)  ← the make-or-break loop

Repeat until a frame presents:
- [ ] Run; capture the crash (address, last log line).
- [ ] Classify with the triage table in `50-cpu-recompilation.md`
      (jump table? boundary? save/restore? longjmp/EH? endianness?).
- [ ] If it's a **missing import** → add a shim/stub (`70-runtime-kernel-and-xam.md`).
- [ ] If it's **structural** → fix config and re-codegen.
- [ ] If it's **surgical** → mid-asm hook / function override (`80-…`).
- [ ] Stand up early runtime: memory/heap, TLS, threads, time, **VFS mount**.
- [ ] Bring the **command processor** to "present a cleared frame", then a draw.
- [ ] Gate: **a window shows the title's first frame / menu** (even if ugly).

## Stage G — Render correctness (Phase 4)

- [ ] Per-shader fixes via the runtime translation path
      (`60-gpu-shader-translation.md`): vertex formats, semantics, instancing,
      `R11G11B10`, alpha test.
- [ ] Render state: blend/depth/raster, RT formats, **EDRAM resolve**, gamma.
- [ ] Textures/samplers: formats, swizzles, filtering, cube maps.
- [ ] Gate: **menus + an in-game scene look correct** vs reference footage.

## Stage H — Make it playable (Phase 5)

- [ ] **Input**: XInput → host controller(s) (+ keyboard); multiple pads for co-op.
- [ ] **Audio**: XMA decode → mixer → host output (`75-…`).
- [ ] **Save/continue**: content APIs → host save dir; verify across relaunch.
- [ ] **Timing**: stable frame pacing; fix timing-dependent gameplay/AI.
- [ ] Gate: **boot → menu → full match → win/lose → save → relaunch → continue.**

## Stage I — Polish, perf, package (Phase 6)

- [ ] Enable codegen **optimizations** (`*_as_local`, `skip_lr`) incrementally;
      re-verify each.
- [ ] Performance pass (CPU hotspots, GPU state churn, shader warm-up).
- [ ] Config/CVars, controller remap, resolution/window options.
- [ ] Package a Release build; write `docs/RUN.md`.
- [ ] Stub online/leaderboards/achievements to graceful "offline".

---

## Go/no-go gates (summary)

| Gate | Pass condition | If it fails |
|---|---|---|
| Feasibility (C) | small import surface, no online core dep | re-scope or pick another title |
| Compile (E) | app links | fix config / unresolved symbols |
| **Boot (F)** | a frame presents | the critical one — keep iterating; this predicts the rest |
| Render (G) | looks correct | per-shader/state debugging |
| Playable (H) | full loop works | subsystem-by-subsystem |

## What to record (so the next title is easier)

At every stage, append to the title's case study:
- `20-imports-backlog.md` — each import: implemented/stubbed/todo.
- `30-boot-log.md` — each crash: address → cause → fix (the most reusable log).
- `40-render-notes.md` — each shader/render fix: hash/symptom → cause → fix.

Then **promote anything general** (a new pitfall, a reusable shim pattern, a
jump-table variant) into `general/95-pitfalls-and-patterns.md` so it benefits all
future ports. That promotion step is what makes this KB compound in value.
