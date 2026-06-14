# In-match lag: root cause, fix, and the v1.0 line

Status as of **2026-06-10 (v1.0, maintainer-accepted)**. This is the performance
finale for the port and **supersedes the "v1 COMPLETE" framing in
`90-progress-report.md`** — that banner is the *bring-up* milestone (playable to
a win, dated 2026-05-24) and contains **no** perf work. The reusable mechanism +
levers live in `general/55`; this doc is the title-specific instance.

## Symptom
The **field** (gameplay) ran in slow-motion from the first seconds of control on
level 1; **menus were fine**. (Earlier "lag scales with enemy count" framings
were wrong — re-verified by actual play, 2026-06-10.)

## Two layers (both root-caused, play-verified)

**Layer A — host CPU-frequency trap. FIXED.** Linux `intel_pstate` EPP=balance
down-clocked the latency-bound (idle-looking) match load, throttling the first
seconds of action. Fixed by persisting **EPP=`performance`** (systemd
`cpu-epp-performance.service`) + a `--freq_keeper` thread (patch **0018**).
Cold-entry now holds **60.0 @ ~2.9 GHz**. Mechanism: `general/55` failure mode 2.

**Layer B — serial-chain translate capacity. Residual, accepted for v1.0.**
The title is a **60-fps design** (paces to the 60 Hz vblank; sim ticks once per
rendered frame — it is NOT "a 30-fps game"). Per-draw CP PM4→Vulkan translate ≈
**8.4 µs/draw**, so the 60-fps ceiling is ≈ **1990 draws/swap**. Level-2+ peak
waves measure **1900–2250 draws/swap**, so the heaviest ~tens of seconds re-grid
to an **exact 30.0 = 2× slow-motion** (sound/input stay normal — own threads).
Mechanism: `general/55` failure mode 1. Playtest: avg **54.2**, max 60.0, min
30.0; maintainer accepted it as **v1.0**.

## Perception note
Maintainer asked "is it sped up now?" — **no.** Pre-fix the field ran ~2.3–2.5×
slow-mo and was learned as "normal", so the corrected native pace reads as fast.
The XE_SWAP vsync limiter + one sim tick/frame cap the game at **≤1× native**; it
cannot fast-forward unless you set `vsync=false` (which then runs sim at render
speed on light scenes = wrong). So "feels fast" ≠ "is fast".

## What shipped (patches 0017–0019)
The perf keepers are 0017–0019 (incl. **0019** eager vblank-fence push = "step 1"
of lever B below). The full granular, byte-verified series is the port's
`patches/README.md`; the 24 closed throughput levers are in
`docs/archive/FLOOR-*` (don't retry — summary table in `general/55`).

## v2 (if resumed) — NOT started
Goal: hold ≥ ~50 effective fps through the heaviest waves of **all** levels.
Step 0 = size level-3+ peaks; then lever A (texture-array draw batching) and/or
lever B (true guest↔CP pipelining). Both are fully specified — **read the port's
`docs/ROADMAP-V2.md` before any perf session; do not re-derive.** A Windows-port
checklist (2 mechanical code gaps + the EPP/freq Linux-only caveat) is in the
same file.
