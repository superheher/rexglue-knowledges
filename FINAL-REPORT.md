# Final report — South Park: Let's Go Tower Defense Play! (XBLA, 58410931) static recompilation

A short, honest capstone for both deliverables: the playable port and the reusable KB.
Companion to the per-subsystem deep dives in `titles/south-park-lgtdp/` and `general/`.

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
- **Save — WRITES to disk in full mode (verified); CONTINUE (cross-restart) — NOT working yet.**
  In full mode the game writes profile/progress to `userdata/58410931/profile/User/63E83FF*`
  (trial wrote 0 bytes); completing Stan's House **unlocks the next level in-session** and rewrites
  the blobs. **But on the next launch the unlock is gone** (level re-locked) — the game reads the
  saved blobs at boot (instrumented: `[PROF-LOAD]`/`[PROF-RD]` fire, `is_set=true`) yet progress
  still doesn't carry across a restart; a write during boot/LOCAL-GAME-start appears to reset it.
  This is the **one remaining Condition-A "continue" gap** (under active diagnosis with
  `[PROF-*]` instrumentation in the runtime). See `55-save-system.md`.
- **Open polish:** in-match font-glyph corruption (front-end text is fine — `65-font...md`); audio
  fidelity (thread runs, not ear-verified).

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

## What remains (precise, actionable) — updated 2026-05-24
The boot deadlock is **RESOLVED** (CP `WAIT_REG_MEM` 1 ms-sleep-per-poll throttle → spin-yield fix;
see Outcome). Driving without focus is done (`REX_INPUT_FILE`/`live_input.txt` + `launch_game.bat`).
The remaining items:

1. **Continue (cross-restart save) — THE remaining Condition-A gap.** Save WRITES work in full
   mode, but progress doesn't survive a restart. Instrumentation (`[PROF-RD]/[PROF-WR]/[PROF-LOAD]/
   [PROF-SAVE]` in `xam_user.cpp` + `user_profile.cpp`) shows the game **does read+load the blobs at
   boot** (`is_set=true`), so it's not a missing load — a write during boot/LOCAL-GAME-start appears
   to **reset** the campaign blob to defaults before the level-select reflects it. Next: capture the
   `[PROF-WR]` that clobbers (which `setting_id`, when) on a restart that has real saved progress,
   and determine whether it's a game reset on "new local game" vs a runtime profile-identity issue
   (does the game associate the save with a stable signed-in XUID?). Candidate fixes once located:
   suppress the default-clobber, or present a stable returning-user profile so the game loads
   instead of re-initializing.
2. **In-match font-glyph corruption** (`65-font-glyph-corruption.md`): front-end/menu text is crisp;
   only the in-match HUD/tooltip text mis-decodes (striped). Fix via the GPU trace + texture dump
   (`trace_gpu_prefix`, `trace_dump.cpp`) to inspect the in-match font texture's tile mode/format.
3. **Audio fidelity:** the XMA thread runs; needs ear verification + any conversion fixes.
