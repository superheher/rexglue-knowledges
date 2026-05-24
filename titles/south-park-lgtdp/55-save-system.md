# The save system — full architecture + why a blind run never triggers it (2026-05-24)

This is the deep characterization of the title's progress-save path, instrumented and
verified live. **Headline: the save subsystem is fully implemented and correctly wired;
it just never *requests* a save in any flow that can be driven blind (tutorial win →
continue, options, exit-to-menu). The save is request-queued + async, and the request is
only enqueued by a game progress/settings event that blind navigation doesn't reach.**

> ## ✅ UPDATE 2026-05-24 — ROOT CAUSE WAS **TRIAL MODE**, and the save now WRITES TO DISK (verified)
> The "never requests a save" framing below was only half the story. The deciding factor is
> the **license**: the title was running as a **TRIAL** because `XamContentGetLicenseMask`
> returns the runtime cvar **`license_mask` (default 0)**. In trial mode the game shows
> "UNLOCK FULL GAME" / "you can keep this achievement once you unlock the full game" and
> **deliberately persists nothing** — that is why *zero* bytes were ever written, no matter
> which flow was driven.
>
> **Fix (no rebuild): launch with `--license_mask=1`** (this title's full-game license; the
> standard owned-game unlock, same cvar Xenia uses). Then, **verified live**:
> - the main menu drops "UNLOCK FULL GAME" (now starts at LOCAL GAME) → full version;
> - the game **writes profile settings to disk** at sign-in: `userdata/<titleid>/profile/User/
>   63E83FFD`,`63E83FFE`,`63E83FFF` (1000 B each), via **`XamUserWriteProfileSettings`**
>   (the recomp.27 site), NOT the `XamContentCreateEx` content path below;
> - changing a value in **HELP & OPTIONS → SETTINGS → ACCEPT rewrites those 3 files
>   immediately** (mtime jumps to now). In trial the same ACCEPT wrote nothing.
>
> So the save **path works end-to-end in full mode**; the trial license was the gate. Lesson
> promoted to general/95 ("a recompiled XBLA title that won't persist is often running as a
> TRIAL — check `XamContentGetLicenseMask`/`license_mask` before chasing the save-enqueue").
>
> ### Where campaign progress actually lives (corrects the chain below)
> The `XamContentCreateEx` content-save chain analyzed below **never fires** even in full mode
> (0 `XamContent*` calls in a full playthrough; no content save-game dir is created). Instead,
> **this title stores its campaign progress in the title-specific PROFILE settings**
> (`XamUserWriteProfileSettings` → the binary blobs `0x63E83FFD/E/F`). Verified live in full
> mode: completing **Stan's House** (waves 1-4) **unlocked the next level (Elementary School)**
> in the level-select, and the profile files were **rewritten on disk during/after the level**
> (mtimes jumped 09:57 → 10:10 → 10:22 as progress advanced). So the `sub_82129730`/
> `XamContentCreateEx` machinery documented below is **not** this game's progress path — it's
> a secondary/unused one. The lever for "does progress save" here is simply **full license +
> the profile-settings write path** (both working). Continue (load-back on restart) uses the
> runtime's `UserProfile::LoadSetting` (reads `profile/User/<id>`, `BinarySetting::Deserialize`
> sets `is_set=true`, and `XamUserReadProfileSettingsEx` returns it) — that path *looks* correct
> in the runtime.
>
> ### ❌ BUT continue (cross-restart) does NOT work — progress resets on restart
> Verified the hard way: completed Stan's House (level 2 unlocked **in-session**, disk files
> rewritten 10:10→10:22), then restarted with `--license_mask=1` and went back to the level
> select → **only Stan's House unlocked again (level 2 re-locked)**, and the profile files were
> **overwritten at boot (10:28:50) with default content**. So on a new boot the game **starts a
> fresh profile and clobbers the saved one instead of loading it** — the in-session unlocks never
> carry across a restart. (Same in-game *audio/subtitle* settings also reset.) So: **SAVE writes
> happen, but CONTINUE (load-back) is broken.** The likely gap is the **profile-load order**: the
> game writes a default/fresh profile during boot/sign-in/LOCAL-GAME before (or instead of)
> reading the saved one — i.e. it never loads `0x63E83FF*` into its in-memory state on boot. To
> fix/confirm needs instrumentation of the read/write order (`XamUserReadProfileSettings` vs
> `WriteProfileSettings` calls for `0x63E83FF*` at boot) — does the game even *read* on boot, or
> only write? Candidate runtime fix: **eagerly load the title-specific settings from disk at
> sign-in** (so `is_set=true` with saved data) before the game can overwrite them — but only
> helps if the game then reads them into its own state. **This is THE remaining Condition-A
> "continue" gap.**

### ✅ DEFINITIVE mechanism (instrumented `[PROF-*]` in the runtime, byte-level proof)
Completing Stan's House sets **`63E83FFE` offsets 746 & 755 → 1** (level-complete flags) + score
bytes in `63E83FFF`; `63E83FFD` is constant (`00 01 02 03…`). Saved to disk. Then on restart:
1. **Boot READS + LOADS the progress** — `[PROF-LOAD] 63E83FFE LOADED 1000 bytes` then
   `[PROF-RD] 63E83FFE is_set=true`, disk still `FFE[745]=1`. The runtime serves the saved
   progress correctly; **no clobber at boot.**
2. **During LOCAL GAME → lobby → CAMPAIGN navigation the game WRITES all three blobs back as
   DEFAULT** — `[PROF-WR] 63E83FFE` → `[PROF-SAVE]`, disk now `FFE[745]=0`. The game's in-memory
   campaign state was **default** at write time → it did **not** apply the boot-loaded progress;
   it **resets on "new local game."** The level-select then reads the just-reset default → only
   Stan's House unlocked.

So the runtime save/load is correct; the gap is **the game not loading its saved progress into the
campaign state (it re-inits + clobbers on LOCAL-GAME entry).** Fix needs game-side RE: log the
guest call-stack at the clobbering `XamUserWriteProfileSettings` to find the reset function, or find
the campaign-load that *should* apply `63E83FFE` on CAMPAIGN entry. **Within a session progress
works (a level-complete unlocks + auto-advances to the next); only cross-restart continue is broken.**

## The chain (endpoint → trigger), all source-verified in the codegen
| Function | Role |
|---|---|
| `sub_824485F8` | **endpoint** — calls `XamContentCreateEx` (the actual save); validates the content struct or bails error 87 |
| `sub_82448698` | thin wrapper over `sub_824485F8` (recomp.30:28153) |
| `sub_82129AE8` | serialize + save (content-type 3) |
| `sub_82129958` | **save-decision** — `(r3=index, r4, r5, r6=&entry)`; writes the entry header then serializes. Called **per queued entry** |
| `sub_82129730` | **async save state machine / pump** — global state at **`0x82919B40`** (states 0→1→2→3→4). Iterates a **queue of 308-byte save-entries** (`[statebase+44]`, count in `r28`) and calls `sub_82129958` for each |
| `sub_82151170` | **flush-if-dirty** — `if ([saveobj+25]==0) return;` else pumps `sub_82129730`. `saveobj = 0x827F4D8C` (fixed global). Called **~35×/s (17 638×/run)** |
| `sub_8215E2E0` | **unconditional save-event** — pumps `sub_82129730` with no dirty gate |
| `sub_8215D348` | **save-handler** that contains all **4** call sites of `sub_8215E2E0`. **No direct callers — invoked indirectly** (registered callback / save worker; only appears in the function table + `SetFunction`) |

So there are two ways the pump (`sub_82129730`) ever runs: (a) the **dirty flag** `[0x827F4D8C+25]` is set, so the per-frame `sub_82151170` stops skipping; or (b) the save-handler `sub_8215D348` runs and calls `sub_8215E2E0`. **Neither happens in a blind run.**

## Live evidence (instrumented, this build)
- `[SAVE2]` on `sub_82151170`: fires **17 638×/run**, but **`dirty[+25]=0` every single time** → the flush always early-returns. The save-data object is **never marked dirty**.
- `[SAVE-DIAG]` on the save-decision `sub_82129958`: **0 fires** in every driven flow (win→continue, HELP & OPTIONS, EXIT GAME) → the entry queue is always empty when pumped.
- `[SAVE3]` on the save-handler `sub_8215D348`: **fires once at boot** with non-guest pointers (`this=0x02C6…`, an init/registration artifact) and **never again** with a real request.
- Forcing the dirty flag true (override `[+25]` for a range of frames) **does** make `sub_82151170` pump `sub_82129730`, but it **still never reaches `sub_82129958`** — because the entry queue is empty + the async state needs a genuine queued request, not just a pump. So forcing the *gate* is the wrong lever; the missing thing is the **enqueue**.
- The write path itself is proven good: the shader cache writes to `user_data_root` fine, and `XamContentCreate*`/dummy storage devices are implemented (see [[50-menu-input-and-lobby]]).

## What this means for Condition A's "save/continue"
The recomp's save is **not broken** — it is **un-triggered**. To make it fire you must reach
the game event that **enqueues a save-entry** (or sets `[0x827F4D8C+25]`). Candidates, none
reachable by the blind injector path used so far:
- completing a **non-tutorial** stage (Stan's House is the tutorial; later stages are locked
  behind progress that itself needs the first save — a bootstrap the tutorial-complete save
  is presumably meant to write);
- a **settings commit** in HELP & OPTIONS (changing a value, then backing out) — navigated to
  but no value was actually changed/committed;
- exit-to-dashboard with unsaved progress.

**Next concrete leads** (for a human play-test or a focused next session):
1. Find the **enqueue**: `sub_8229BFE8` (recomp.13, heavy 308-byte-stride use — the save-entry
   array manager) and the `sub_8229Bxxx` family (`sub_8229BBE8` init, `sub_8229BD90`). Find who
   adds entries to `[0x82919B40+44]` and increments the count.
2. Find what **indirectly invokes** `sub_8215D348` (it's a registered worker/callback) and what
   posts requests to it — that's the explicit "save now" path.
3. Or simply **play to a real save point**: the `[SAVE-DIAG]`/`[SAVE3]` instrumentation is left
   in this build, so a human playthrough will print `[SAVE-DIAG] … (SAVING)` the instant the
   game saves — pinpointing the trigger with zero extra work.

## Reusable lesson (promoted to general/95)
Recompiled console saves are often **async + request-queued state machines**, not synchronous
`fwrite`s. A per-frame "flush if dirty" that you see firing thousands of times with the dirty
flag always 0 is **not** the trigger — it's the *pump*. Trace to the **enqueue** (who marks
dirty / who posts the request), and don't be fooled into "forcing" the gate: that pumps an
empty queue. Leave the endpoint (`XamContent*`) instrumented so a real playthrough self-reports.
