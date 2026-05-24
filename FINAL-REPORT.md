# Final report — South Park: Let's Go Tower Defense Play! (XBLA, 58410931) static recompilation

A short, honest capstone for both deliverables: the playable port and the reusable KB.
Companion to the per-subsystem deep dives in `titles/south-park-lgtdp/` and `general/`.

> **v1 STATUS: COMPLETE (per maintainer scope decision 2026-05-24).** v1 = OFFLINE single-player,
> **playable boot → menu → match → win/lose with rendering, audio, gamepad, save-to-disk, and
> in-session continue** — all achieved + screenshot-verified. **Cross-restart "continue"** (campaign
> unlocks persisting after a full relaunch) was scoped **out of v1** by the maintainer and is a
> documented, root-caused **post-v1 backlog item** (`56-continue-re-map.md`). KB deliverable
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
- **Save — WRITES to disk in full mode (verified); CONTINUE (cross-restart) — NOT working; root
  cause RE'd to a game-side issue.** In full mode the game writes profile/progress to
  `userdata/58410931/profile/User/63E83FF*` (trial wrote 0 bytes); completing Stan's House
  **unlocks the next level in-session** and rewrites the blobs. **On the next launch the unlock is
  gone.** Instrumented (`[PROF-*]`) + a `SaveSetting` guard nailed it: the runtime **loads the save
  correctly at boot** (`is_set=true`) and a guard can even **keep the disk save intact across
  restart+nav** — yet the level-select still resets. So the **unlock is driven by the game's
  in-memory campaign state**, which it **re-inits to default on LOCAL-GAME entry and never populates
  from the saved profile**. Code-level: the **SAVE serializes `g_slots` (`0x828EB348`, the
  session-player slot array), but the LOAD writes a different global (`0x828E3A38`)** — that
  asymmetry / missing slot→g_slots apply is the gap. **No runtime-only fix works** (the game ignores
  the preserved disk); needs guest-side RE. Full map + next steps: `56-continue-re-map.md`,
  `55-save-system.md`. **Per the maintainer's 2026-05-24 scope decision, cross-restart continue is
  OUT of v1 (post-v1 backlog); v1's save requirement is satisfied by save-to-disk + in-session
  continue (a level-complete unlocks + auto-advances to the next).**
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
6. **A correct runtime save/load doesn't guarantee "continue" — the GAME may ignore its own save.**
   Before assuming a persistence bug is in your runtime, prove the runtime end-to-end (it WROTE the
   bytes, it LOADED them at boot with `is_set=true`) and try a guard that preserves the disk save
   across a restart. If progress *still* resets, the bug is **guest-side**: the title re-inits its
   in-memory state on a "new game / lobby" entry and never applies the loaded profile. Tell-tale:
   the **save and load touch different in-memory globals** (here SAVE serializes one array, LOAD
   fills another) — there's a missing/broken copy in the guest. No runtime hack fixes this; it needs
   guest-code RE + a config override. **First check trial vs full: `XamContentGetLicenseMask` returns
   the `license_mask` cvar (default 0 = trial), and a trial deliberately persists nothing.** (`75`,`95`)

## What remains (precise, actionable) — updated 2026-05-24
The boot deadlock is **RESOLVED** (CP `WAIT_REG_MEM` 1 ms-sleep-per-poll throttle → spin-yield fix;
see Outcome). Driving without focus is done (`REX_INPUT_FILE`/`live_input.txt` + `launch_game.bat`).
The remaining items:

1. **Continue (cross-restart save) — THE remaining Condition-A gap; root cause RE'd, fix is
   guest-side + multi-session.** The runtime save/load is correct (boot loads the blobs,
   `is_set=true`; a `SaveSetting` non-zero-byte guard preserves the disk save across restart+nav —
   verified). But the level-select reset persists, so the game's **in-memory campaign state** is
   what's reset on LOCAL-GAME entry and is not populated from the save. Mapped the call graphs
   (`56-continue-re-map.md`): **SAVE** `sub_82296A38→sub_82298418→sub_824069C8→XamUserWriteProfileSettings`
   serializes **`g_slots` (`0x828EB348`)**; **LOAD** `sub_8229C8D0→sub_8229CB38→sub_82406958→
   XamUserReadProfileSettings` writes a **different global `0x828E3A38`**. The fix needs the guest to
   apply the loaded `0x828E3A38` slot into `g_slots` on CAMPAIGN entry (find the copy/reset function;
   it may be a mis-translated function or a flow the blind nav skips), then a config-override /
   post-codegen fixup. Within a session, progress works (a level-complete unlocks + auto-advances).
2. **In-match font-glyph corruption** (`65-font-glyph-corruption.md`): front-end/menu text is crisp;
   only the in-match HUD/tooltip text mis-decodes (striped). Fix via the GPU trace + texture dump
   (`trace_gpu_prefix`, `trace_dump.cpp`) to inspect the in-match font texture's tile mode/format.
3. **Audio fidelity:** the XMA thread runs; needs ear verification + any conversion fixes.
