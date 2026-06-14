# Performance & pacing of a recompiled title

How a static-recompiled 360 title actually spends each frame, why the usual
throughput optimizations often do **nothing**, and the two failure modes unique
to a *paced* guest: **time dilation** (vsync slow-motion) and **host
power-management throttling**. Method here is title-agnostic; the absolute
numbers come from the running case study (`titles/south-park-lgtdp`, the rexglue
runtime over Vulkan).

> Cite: empirical values below are measured on *South Park: LGTDP* on one
> Linux/Vulkan host (one RTX-class GPU). Treat the **shape** as general and
> re-measure absolute numbers per title/host. Source: the port's
> `docs/ROADMAP-V2.md` + `docs/archive/FLOOR-*` (24 closed-lever reports).

## The per-frame chain is SERIAL and LATENCY-bound

A recompiled title is **not** the "many cores busy" workload people expect from
an emulator. Per displayed frame the work is a single **dependency chain**:

```
guest sim tick ─▶ command-processor translate (guest PM4 ─▶ host Vulkan/D3D,
                  ONE thread) ─▶ present/swap ─▶ guest waits its own render
                  fence ─▶ next sim tick
```

- The guest **waits on its own render** (an in-stream fence) before the next
  tick, so sim and translate do **not** overlap by default — frame time is the
  *sum* of the stages (a latency), not a throughput you can hide with more
  cores. Profiles show the other cores ~idle and one thread on the critical
  path. (CP: `75`; the recompiled guest code: `50`.)
- The dominant, scaling cost is **command-processor translation**, paid **per
  draw**. Measured here ≈ **8.4 µs/draw** (after redundant-register-write
  elision). The 60-fps ceiling is then `16.7 ms ÷ per-draw` ≈ **~1990
  draws/swap** on this host; the heaviest waves hit 1900–2250 and blow it.

**Diagnose the regime before optimizing.** If a perf floor will not move under
the throughput levers below, it is **latency-bound** — stop making the busy
thread faster in bulk and attack the *chain*.

## Failure mode 1 — vsync time-dilation (it's "slow-motion", not "lag")

A title that (a) paces to vblank (vsync-limited swap) and (b) ticks its sim
**once per rendered frame** has a cliff: the instant per-frame work overruns the
refresh interval, the swap can only land on the **next** vblank, so the
effective rate snaps to an **exact integer divisor** of refresh — 60 → **30.0**,
never "52-ish". Because sim advances once per frame, **game time slows in
proportion**: 30.0 fps = exact **2× slow-motion**. Not tearing, not random jank.

| Tell | Reading |
|---|---|
| Frame interval | **grid-pure** at the divisor (clean 30.0 / 20.0), not noisy |
| Audio / input | run at **normal speed** (own threads) while *video* slows |
| Player feel | "everything moves in slow-mo", usually misreported as "laggy" |
| `vsync=false` probe | flips slow-mo → **fast-forward** (sim then runs at render speed on light scenes) — proves the coupling, **not** a fix |

**Fix = make a frame fit, or remove the cliff:** cut per-draw cost (lever A) or
pipeline guest↔CP and advance the swap fence at ring-arrival so a >16.7 ms frame
degrades to an honest 40–56 fps instead of snapping to 30.0 (lever B). See the
two levers below + the title's `75`.

> ⚠️ Perception trap: if the title ran slow (un-fixed) for a while, players
> learn the slow pace as "normal" and the **correct** native speed then reads as
> "sped up". Confirm pace against the divisor grid + audio, not by feel.

## Failure mode 2 — host CPU power-management throttles the idle-looking load

Because the chain is latency-bound, the busy thread leaves the **cores mostly
idle** between fence waits. A modern governor reads that as "not busy" and
**down-clocks** exactly when you need peak single-thread speed — so the title
throttles in the **first seconds of action** and looks like a code regression.

- Linux `intel_pstate` with **EPP=`balance_performance`** / a schedutil governor
  is the usual culprit; the analog elsewhere is the OS power plan.
- **Fix:** pin **EPP=`performance`** persistently (a systemd unit) and,
  optionally, an in-process "freq keeper" thread that nudges the governor while
  the match runs. On this title that erased the first-seconds throttle
  (cold-entry then held native clocks). Windows: set the high-performance power
  plan — the trap may not exist there at all.
- **Tell:** clocks sag (`turbostat`/`perf`) under light *measured* CPU%; forcing
  the governor to performance removes the dip with **no code change**.

## Throughput levers that do NOT move a latency-bound floor

Tried and closed on this title (full reports: the port's `docs/archive/FLOOR-*`).
Recorded so the next port doesn't re-spend the weeks:

| Lever | Result on a latency-bound floor |
|---|---|
| PGO, BOLT, ThinLTO, ICF | floor-neutral |
| Leaf-inlining, codegen-size tuning | floor-neutral |
| `-mcmodel` beyond `medium` | neutral (but see the `large` **bug** below) |
| Removing busy-wait spins | fixed a *waste*, not the floor |
| GPR-as-local / register promotion | NO-GO / neutral |
| **Order-safe** draw batching (adjacent same-state) | ≈ neutral (avg run length ~1.36) |
| Moving translation off the critical thread | neutral while the guest still waits its own fence |

The one keeper from the codegen pass was a **bug fix**, not a tuning win:

- **`-mcmodel=large` → indirect-call storm.** A large recompiled `.so`/DLL built
  with the *large* code model routes nearly every call through a GOT/indirect
  thunk; the indirect-call volume tanks throughput. **`-mcmodel=medium`** cut
  indirect calls ~**−78%** and lifted the floor. Linux GCC/Clang only — MSVC's
  model never had this pathology, so guard the flag `if(NOT WIN32)`. (`50`)

## The two real levers (when you ARE latency-bound)

1. **Cut per-draw translate cost** — batch draws *across* texture/state switches
   via **texture arrays / bindless**. (Order-safe batching of adjacent same-state
   draws alone is ~neutral; the ~7× headroom is in merging across the switches
   that break groups.) Raises the draw ceiling for every level; no pacing
   risk. Gate with screenshots + a determinism diff (translation output must not
   change). (`60`, `75`)
2. **Pipeline guest↔CP + swap-at-ring-arrival** — double-buffer the
   guest-visible GPU state the CP reads (register snapshot / shadow state) so sim
   frame N+1 runs while the CP still translates frame N, and advance the guest's
   vblank/swap fence when the swap *arrives in the ring* (not when present
   completes). Removes the 2-vblank cliff (failure mode 1). Costs +1 frame of
   input latency; watch state hazards (a global write-elision cache must become
   per-frame) and the boot first-present fence path. (`75`, `70`)

## Measuring perf changes (so a "win" is real)

- Bench **natural play** at the heaviest wave, not a synthetic loop; log per-swap
  **draw count** + frame interval (a pacing-diag counter) so grid-purity (failure
  mode 1) is visible directly.
- Hold an **A/B keep-bar** across repeats (median **and** worst-case must not
  regress) and a **determinism gate** whenever the translation path changes.
- Separate **cold** (first-boot shader/pipeline translation — one-time, cached)
  from **steady-state**; never "optimize" a one-time cold cost. (boot entries in
  `95`)

---
Concrete case study: `titles/south-park-lgtdp/75-in-match-lag-and-perf.md` (the
two-layer diagnosis + v1.0 outcome). Quick triage: `95` → "Performance / pacing".
