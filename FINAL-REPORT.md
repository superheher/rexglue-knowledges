# Final report — South Park: Let's Go Tower Defense Play! (XBLA, 58410931) static recompilation

A short, honest capstone for both deliverables: the playable port and the reusable KB.
Companion to the per-subsystem deep dives in `titles/south-park-lgtdp/` and `general/`.

> **v1 STATUS: COMPLETE (per maintainer scope decision 2026-05-24).** v1 = OFFLINE single-player,
> **playable boot → menu → match → win/lose with rendering, audio, gamepad, save-to-disk, and
> in-session continue** — all achieved + screenshot-verified.
> **✅ UPDATE 2026-05-24: CROSS-RESTART CONTINUE NOW WORKS too (post-v1, verified by running).**
> Win Stan's House → restart → the CAMPAIGN LEVEL SELECT shows Elementary School unlocked. Root
> cause: campaign progress lives in the in-memory g_slots block `0x828EB348+2480..+2624`; the guest
> reads `63E83FFE` at boot but never deserializes it into that block, and the game clobbers the disk
> save. Fixed in the runtime (SDK patch): a 20 ms background thread snapshots the g_slots progress
> block to a side file on a win, and `XamInputGetState` restores it into g_slots each frame so the
> level grid is rebuilt with the saved unlocks. Format-agnostic + monotonic; the decisive RE was a
> live ReadProcessMemory before/after diff across an in-session unlock. Full analysis +
> verification: `56-continue-re-map.md` (SOLUTION). KB deliverable
> complete (this report + per-subsystem docs + promoted general lessons + templates).

## Outcome (honest) — updated 2026-05-24
- **Playable single-player from boot to a win — achieved and screenshot-verified** (rexglue-sdk):
  `boot → intro → title (PRESS START) → main menu → LOCAL GAME → lobby → CAMPAIGN → CASUAL →
  level select (Stan's House) → the MATCH (HUD, waves, combat) → "STAGE COMPLETE!" (win) →
  CONTINUE`. **Rendering correct, input works all the way through (driven WITHOUT window focus via
  `REX_INPUT_FILE`/`live_input.txt`), XMA audio thread runs, no crash.** Covers Condition A's
  "boot → menu → match → win/lose: rendering, audio, gamepad".
- **Boot deadlock — RESOLVED by a code fix (no reboot).** The earlier "environment/reboot-only"
  conclusion was WRONG. Root cause: the command processor's `WAIT_REG_MEM` poll loop slept a fixed
  **1 ms per unmatched poll** under vsync → the CP starved and froze at frame 1. Fix: **spin-yield
  (`SyncMemory`+`MaybeYield`) ~8000 polls before any sleep** (+ raise the per-poll log threshold).
  Boots to the title every run now. See `60-boot-present-deadlock.md` / general/95.
- **The title runs as a TRIAL by default** (`XamContentGetLicenseMask` returns the `license_mask`
  cvar, default 0). Trial persists nothing ("UNLOCK FULL GAME"). **Run `--license_mask=1`** (owned
  copy) → full version.
- **Save — WRITES to disk in full mode (verified); CONTINUE (cross-restart) — ✅ NOW WORKING.**
  In full mode the game writes profile/progress to `userdata/58410931/profile/User/63E83FF*`
  (trial wrote 0 bytes); completing a level **unlocks the next in-session** and the unlock now
  **persists across a restart**. The fix wasn't on the disk-save path (the game re-inits its
  in-memory campaign state on LOCAL-GAME entry and never deserializes the saved profile into it):
  it was finding **which in-memory global the level grid is built from** — the `g_slots` block
  `0x828EB348 + 2480..+2624` (gate byte `+2520`) — via a **live ReadProcessMemory before/after
  diff** across an in-session win, then having the runtime **snapshot/restore that block**
  (20 ms background thread → `gslots_campaign.bin` on a win; `XamInputGetState` restores it each
  frame when the live block is default). Format-agnostic + monotonic. Verified by running: win
  Stan's House → restart → Elementary School unlocked. Full analysis: `56-continue-re-map.md`
  (SOLUTION); the "the game ignores its own save" earlier verdict was too pessimistic.
- **Resolved polish (all verified by running):** in-match font-glyph corruption (GPU
  shared-memory page-validity race — `65-font...md`); Elementary `en-en` campaign slides/diagrams
  (runtime locale-subdir fallback — `67-polish-backlog.md`); `--always_win` invincibility cheat
  (`66-...md`); boot speed (profiled ~50–60 s warm; the old "4–5 min" was a cold shader cache —
  `67`). **Open (needs a human / out of scope):** audio fidelity (path objectively correct +
  clips produced; ear sign-off pending — `67`); intro/cutscene WMV movies black & silent
  (no WMV3/WMA2 decoder; user-skippable; documented limitation — `67`); online co-op/leaderboards
  /achievements/avatars (out of scope for v1, stubbed offline).

## What worked
- **rexglue-sdk** as the primary toolchain: codegen → config-driven fixups → D3D12 runtime. The
  guest-correctness wins were almost all **config + a post-codegen fixup script**, not hand-edits.
- **Live instrumentation over guessing.** Every hard bug was cracked by adding a targeted log to a
  generated function (or `cdb` on the live process), watching it run, then fixing root cause:
  the image-load `setjmp/longjmp`, the session-enroll null-deref class, the save architecture, and
  the boot deadlock were all settled empirically.
- **Two decisive root fixes:** (1) the image-format-detection **custom setjmp/longjmp** modeled via
  `setjmp_address=0x8242EEA0`/`longjmp_address=0x8242EA70` (config-only) → reached the title; (2)
  **session-enroll** — route the local signed-in player through `sub_82297F30 loc_82298008` so
  session-player queries stop null-dereffing (`fix_recomp_labels` Fix 6) → reached the match.
- **Cumulative, idempotent post-codegen tooling** (`gen_missing_funcs.py` KNOWN_COMPUTED for
  cross-function branch targets; `fix_recomp_labels.py` for emitted-but-uncompilable constructs).

## Real effort (where the time actually went)
- The long pole was **not** raw recompilation — it was **runtime-behavior bugs** surfaced only by
  running: the post-render hang (a custom EH mistaken for `.xdata` SEH; days of dead-ends before
  the live trace nailed it), the lobby→match null-deref *class*, and GPU-sync/present.
- **Input automation was a tax:** synthetic OS keys never reach SDL here, so an env-gated injector
  was needed just to drive menus; a real focused pad/keyboard works.
- **GPU sync is the recurring hard area:** a non-deterministic guest-GPU-fence stall, then the
  present/vsync boot deadlock. These dominate the remaining risk.

## Top reusable lessons (promoted to general/95)
1. **Console saves are async request-queued state machines.** A per-frame "flush if dirty" you see
   firing thousands of times with the flag always 0 is the *pump*, not the trigger — trace the
   *enqueue*; don't "force" the gate (you pump an empty queue). Leave the `XamContent*` endpoint
   logged so a real playthrough self-reports.
2. **Custom hand-rolled `setjmp/longjmp` ≠ `.xdata` SEH.** If a title does its own image-format
   detection, find the true `setjmp/longjmp` pair by tracing the *caller*, not imports.
3. **"Unresolved call to 0x…" = a cross-function branch target** — register it; keep the generator
   cumulative/idempotent so re-runs don't regress.
4. **A boot freeze at frame 1 that looks "host-state / reboot-only" can be a CP poll-throttle.**
   A fixed per-poll `Sleep` in a GPU busy-wait (`WAIT_REG_MEM`) starves the command processor so it
   falls behind guest frame pacing and freezes — and it looks intermittent across runs (load changes
   the poll count), which tempts a "degraded driver, reboot" conclusion. **Spin-yield before
   sleeping** (and don't log per-poll). Diagnose with `cdb ~*k` + a spin counter; `--vsync=false`
   is a probe, not the fix. (We wrongly concluded "reboot-only" for ~20 cycles; it was a code bug.)
5. **Verify by running, with live instrumentation** — it beats static reasoning for runtime bugs,
   and it keeps you honest about what's actually demonstrated vs. merely plausible.
6. **A correct runtime save/load doesn't guarantee "continue" — the GAME may ignore its own save.**
   Before assuming a persistence bug is in your runtime, prove the runtime end-to-end (it WROTE the
   bytes, it LOADED them at boot with `is_set=true`) and try a guard that preserves the disk save
   across a restart. If progress *still* resets, the bug is **guest-side**: the title re-inits its
   in-memory state on a "new game / lobby" entry and never applies the loaded profile. Tell-tale:
   the **save and load touch different in-memory globals** (here SAVE serializes one array, LOAD
   fills another) — there's a missing/broken copy in the guest. No runtime hack fixes this; it needs
   guest-code RE + a config override. **First check trial vs full: `XamContentGetLicenseMask` returns
   the `license_mask` cvar (default 0 = trial), and a trial deliberately persists nothing.** (`75`,`95`)

## What remains — updated 2026-05-24 (post polish-pass)
The bring-up blockers are all **RESOLVED** (boot deadlock → spin-yield, doc 60; lobby→match
null-deref → session-enroll fix; image-load EH → setjmp/longjmp config). Focus-free driving is
done (`REX_INPUT_FILE`/`live_input.txt` + `launch_game.bat`). The **polish backlog has been worked
to completion** (all verified by running unless noted):

**Done this pass (`67-polish-backlog.md`):**
1. **Cross-restart continue — ✅ SOLVED** (runtime g_slots snapshot/restore; `56`). Win → restart →
   next level unlocked.
2. **In-match font-glyph corruption — ✅ SOLVED** (GPU shared-memory page-validity race fixed under
   the global lock; `65`). "GINGER ▦▦" → "GINGER KIDS".
3. **Elementary `en-en` asset gap — ✅ SOLVED** (runtime locale-subdir fallback in `NtCreateFile`;
   `67 §1`). 8 fallback hits, 0 failures, the school diagram renders. (`School.lua` is a benign
   absent optional script; Elementary plays without it.)
4. **Boot speed — ✅ PROFILED** (`67 §3`): ~50–60 s warm to title; the old "4–5 min" was a one-time
   cold shader-cache translation. CP fence loop healthy (0 stuck). No cheap runtime win remains;
   the residual is game-paced, user-skippable intro animation.

**Open — needs a human or out of scope:**
5. **Audio fidelity** (`67 §2`): the XMA→SDL path is objectively correct (proper 5.1→stereo
   downmix; real-time; no clipping) and **lossless capture clips are produced** (`audio_dump`
   cvar). **Subjective fidelity needs a human ear** on a real audio device — an agent cannot
   ear-verify. (A 3× SDL drain seen in the headless capture is an environment artifact, not a port
   bug.)
6. **Intro / cutscene WMV movies** (`67 §4`): black & silent — the runtime has no WMV3/VC-1 +
   WMA2 decoder (XMA only). The files open and are user-skippable ("Ⓐ SKIP"); the boot doesn't
   block on them. Adding a decoder is a large feature beyond v1 — **documented limitation
   (won't-fix for v1)**.
7. **Online co-op / leaderboards / achievements / avatars** — **out of scope for v1**, left
   stubbed offline.
