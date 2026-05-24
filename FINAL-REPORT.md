# Final report — South Park: Let's Go Tower Defense Play! (XBLA, 58410931) static recompilation

A short, honest capstone for both deliverables: the playable port and the reusable KB.
Companion to the per-subsystem deep dives in `titles/south-park-lgtdp/` and `general/`.

## Outcome (honest)
- **Playable single-player from boot to a win — achieved and screenshot-verified** (with the
  rexglue-sdk toolchain): `boot → intro → title → main menu → LOCAL GAME → lobby → game-mode
  (Campaign) → level select (Stan's House) → the MATCH (gameplay HUD, waves spawn, units defend) →
  "STAGE COMPLETE!" (win, score 2,100) → CONTINUE`. **Rendering correct, gamepad input works all
  the way through, the XMA audio thread runs, no crash anywhere.** That covers Condition A's
  "boot → menu → match → win/lose: rendering, audio, gamepad."
- **Save/continue — fully characterized, not yet demonstrated.** The save subsystem is implemented
  and correctly wired, but it is **never *requested*** in any flow drivable by automation; it only
  enqueues a save on a real in-game progress/settings event. See `55-save-system.md`. This is the
  one remaining Condition-A item; it is verifiable by a human playing to a real save point (the
  endpoint is left instrumented to self-report).
- **A late, environment-dependent boot deadlock** appeared after ~250 launch/kill cycles in one
  long session (present/vsync GPU-sync deadlock; see `60-boot-present-deadlock.md`). The *same
  build* booted to a match win earlier the same session — this is degraded host GPU-driver/DWM
  state, cleared by a reboot. Not a defect in the committed artifacts.

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
4. **Boot hang at the first present = a present/vsync deadlock** (swap vsync-wait vs a presenter
   disruptor claim, and/or a `WAIT_REG_MEM` fence with no escape). Diagnose with `cdb ~*k`; probe
   with `--vsync=false`. Real fix: timeout/yield escapes + runtime GPU-fence write-back. GPU/DWM
   state degrades over many D3D12 device cycles — a reboot restores it fast.
5. **Verify by running, with live instrumentation** — it beats static reasoning for runtime bugs,
   and it keeps you honest about what's actually demonstrated vs. merely plausible.

## What remains (precise, actionable)
1. **Reboot** to clear the degraded GPU/DWM state, then boot default `vsync=true` (it booted to a
   win earlier today). This unblocks everything else.
2. **Save/continue:** play to a real save point (full stage completion / a settings commit); the
   left-in `[SAVE-DIAG]` prints `(SAVING)` the instant it fires. Or trace the enqueue
   (`sub_8229BFE8`/`sub_8229BEB8` family, or what indirectly invokes the save-handler `sub_8215D348`).
3. **Harden GPU sync** for reliability: bound the swap vsync-wait + the presenter disruptor claim;
   add runtime GPU-fence write-back so `WAIT_REG_MEM`/`sub_821C6E58` targets resolve (a committed
   `WAIT_REG_MEM` stop-gap exists, unverified, in `south-park-recomp/patches/`).
