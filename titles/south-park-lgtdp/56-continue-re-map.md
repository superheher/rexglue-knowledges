# Continue (cross-restart save) — reverse-engineering map (2026-05-24)

> **Status: ✅ SOLVED 2026-05-24 — cross-restart continue WORKS, verified by running.** Win Stan's
> House → restart → the CAMPAIGN LEVEL SELECT shows Elementary School UNLOCKED (screenshots
> `C:\Temp\b6_levelselect2.png`, `b11_FINAL.png`). A/B proven (no side file → still locked,
> `b7_levelselect2.png`). Fix is a runtime g_slots snapshot/restore; details in the SOLUTION
> section directly below. The original RE map (mostly superseded/corrected) follows it.

## ✅ SOLUTION (2026-05-24) — runtime g_slots snapshot/restore

**Where the unlock state actually lives (corrects every earlier model in this doc):**
The campaign progress (per-level complete/unlock + scores) is the **active player's slot-0 block at
`g_slots + 2480 .. + 2624`** (g_slots = `0x828EB348`). Diff-proven: winning Stan's House changes ~a
dozen int fields there — notably a **"levels reached" gate byte at `g_slots+2520`** (1 = Stan's only,
3 = Elementary unlocked). The block is **all small ints/flags, NO pointers** (the `0x1e 0x32 0x32`
score thresholds at +2488 match disk `63E83FFF`). The level-select **rebuilds its level grid from
this block** when the CAMPAIGN screen loads.
- **NOT** `g_slots[+0..+20]` (a red herring — `g_slots[+2]` is some minor flag that does NOT drive
  the display; setting it had no effect — verified).
- **NOT** the settings cache `0x828E3A38` (that's the generic per-(user,setting) cache; the level-
  select reads its `[+20]` pointer to the *per-boot heap* level array, which is itself built from the
  g_slots block — so the heap array is downstream, g_slots is the source).

**Why it was broken (root cause):** Two independent game-side faults:
1. The guest **reads `63E83FFE` at boot** (`[PROF-RD] id=63E83FFE is_set=true` fires) but **never
   deserializes it into the g_slots block** — so the in-memory campaign state stays default.
2. The game **clobbers** the disk save to DEFAULT during nav/save (verified: after a win, `[PROF-WR]`
   wrote `63E83FFE` with the level-complete flags back to 0). So even the disk is not a reliable
   persistent source.
The g_slots block is also **transient**: it only holds the won progress around a level win (the
post-win "SCRAPBOOK UPDATED" screen); it reads 0 at the menu/level-select until the grid is built.

**The fix (`third_party/rexglue-sdk/src/kernel/xam/xam_user.cpp` + `xam_input.cpp`, in
`patches/rexglue-sdk-current-full.patch`):**
- A **background thread** (started lazily from `XamInputGetState`) polls `g_slots+2520` every 20 ms;
  when the gate **grows** (a win), it captures the 144-byte block `g_slots+2480..+2624` to a side
  file `userdata/<title>/profile/User/gslots_campaign.bin`. The high-freq poll catches the transient
  post-win window (the player normally lingers on the STAGE-COMPLETE/SCRAPBOOK screen for >1 s, so
  20 ms is ample; the game's own profile-write is useless because it clobbers).
- **Per frame** (`XamInputGetState` → `CampaignContinueRestore`), if the live gate is **lower** than
  the saved snapshot (fresh boot / menu / default block), **copy the saved block back into g_slots**
  — so the level-select grid is rebuilt with the saved unlocks. gate-gated ⇒ monotonic: in-session
  wins (gate ≥ saved) are kept and grow the snapshot; never reduces progress.
- **Format-agnostic** (raw bytes, no need to RE the serialize format) and self-bootstrapping. The
  side file is the source of truth for the restore (replacing the game's clobbered disk save).

**Verified by running (full version, `--license_mask=1`):**
- Win Stan's House (CASUAL) → STAGE COMPLETE → restart → **Elementary School UNLOCKED** in the level
  grid (`b6_levelselect2.png` with bootstrapped genuine win data; `b11_FINAL.png` with production
  thread code). The background thread auto-creates the side file (`[CONT-FIX] saved … (gate=…)`).
- **A/B control:** delete the side file, same build, disk still has progress → **Elementary LOCKED**
  (`b7_levelselect2.png`) ⇒ the side-file restore is what unlocks it (the disk alone does not).

**Live RE technique that cracked it (KB-worthy, promoted to general/):** ReadProcessMemory on the
running recomp at host `0x100000000 + guest_addr` (guest space is mapped at host 4 GB) to dump/diff
guest globals **without a rebuild** — a before/after diff across an in-session unlock pinpointed the
exact bytes. Earlier static-analysis "final models" in this doc were repeatedly wrong; the live diff
was decisive. (Tooling: `C:\Temp\rpm.ps1`.)

---

### (Superseded) original RE map below
> **Status: POST-v1 BACKLOG** (maintainer scope decision 2026-05-24). v1 ships with save-to-disk +
> in-session continue; cross-restart persistence is deferred. This doc is the resumable RE map for
> when it's picked up. (Most conclusions below were CORRECTED by the live RE above — kept for history.)

Resumable RE notes for the (post-v1) cross-restart continue gap. **Symptom:** the runtime saves+loads the
profile correctly (verified: `[PROF-LOAD]`/`[PROF-RD]` fire at boot, `is_set=true`; a `SaveSetting`
guard can even keep the disk save intact across restart+nav), **but the level-select still shows
only Stan's House after a restart** — so the game does not apply the loaded save to the in-memory
campaign state that drives the level-select; it re-inits to default on LOCAL-GAME entry. See
[[55-save-system]] for the disk-level proof. This file maps the guest code toward a fix.

## Call graphs (generated/default/*.cpp, guest addrs)
**SAVE (writes the `0x63E83FF*` profile blobs):**
- `sub_82296A38` (save mgr, recomp.12:71515)
  → `sub_82298418` (recomp.12:75371 — builds an array of **40-byte setting entries**, count =
    `(end-start)/40`, then calls the wrapper)
  → `sub_824069C8` (recomp.27:26719 — wrapper: `XamUserGetXUID` into `[r1+80]`, then
    `XamUserWriteProfileSettings(user=0, count, settings, overlapped)`)
  → `__imp__XamUserWriteProfileSettings`

**LOAD (reads them back):**
- `sub_8229C8D0` (load mgr, recomp.13:7972) — gets the slot via `r29 = sub_8229CA28(key=r3)`;
  if `r29==0` returns; else loads into it via `sub_8229CB38(r29)`.
  → `sub_8229CB38` (recomp.13:8332 — sets up the read request, calls the wrapper, then
    `sub_822A2150` processes the result into `[r29+52]`, stores marker `[r29+56]`)
  → `sub_82406958` (recomp.27:26653 — wrapper: **GATED** — calls `sub_8244E030`; if it returns
    non-zero the read is SKIPPED. At boot it returned 0, so the read proceeded.)
  → `__imp__XamUserReadProfileSettings`

**Two key globals found (both in the 0x828Exxxx session/save region):**
- **LOAD target `0x828E3A38`** — `sub_8229CA28` (recomp.13:8176, `lis -32114; addi 14904`) iterates
  this array searching entries (key at `[entry-60]`, 64-bit = XUID?) for one matching `r3` and
  returns it; the load writes the profile into that **per-user save-slot**.
- **SAVE source `0x828EB348`** — `sub_82298418` (recomp.12, `lis -32113; addi -19640`) iterates
  this array (entries at base+2480, fields `[+0]`=active-flag checked ==1, `[+4]`=id checked !=997)
  and serializes it into the `0x63E83FF*` profile blobs. **`0x828EB348` is `g_slots`** — the
  **session-player slot array** from the lobby/session-enroll fix (slot state at `[slot+2388]`,
  session-player obj at `[slot+2488]`; see [[50-menu-input-and-lobby]] / [[south-park-recomp-progress]]).
  So **the campaign save serializes the session-player slots**, and the campaign progress/unlocks are
  entangled with the session/lobby slot state.

⇒ **The save reads `0x828EB348` (g_slots) but the load writes `0x828E3A38` — DIFFERENT globals.**
That asymmetry is the prime suspect: either `0x828E3A38` is a staging area that must be copied into
`0x828EB348` on CAMPAIGN entry (and that copy is missing/broken), or the two are linked (slot →
data pointer) and the link isn't followed. The level-select unlock reads the g_slots-derived state
(`0x828EB348`), which is re-init'd to default on LOCAL-GAME entry and never populated from the loaded
`0x828E3A38` slot — hence the reset. **Next: confirm by instrumenting reads/writes of both globals
at boot vs level-select, and find the function that should copy `0x828E3A38`→`0x828EB348`.**

## The open question (next session starts here)
The load populates the `0x828E3A38` save-slot, yet the level-select shows level 1. So either:
1. **The level-select unlock reads a DIFFERENT global** (an in-memory "campaign progress" struct),
   and the copy `save-slot(0x828E3A38) → campaign-progress` does NOT run on CAMPAIGN/LOCAL-GAME
   entry (it re-inits to default instead). **Most likely.** → Find the level-select-unlock read
   (what global/field decides a level icon is locked vs unlocked) and the function that should
   populate it from `0x828E3A38`.
2. **Key mismatch** — `sub_8229CA28` is keyed by XUID (`[entry-60]`); if the boot-load key differs
   from the level-select key, they hit different slots. Check `XamUserGetXUID` stability across the
   boot vs the lobby sign-in (the lobby is "1/4 SIGNED IN").
3. The loaded `[slot+52/56]` is a handle/marker, not the unlock flags (unlock processed elsewhere).

## Additional read paths — `sub_824C8660` RULED OUT
`sub_824C8660` (recomp.34:54465) reads with **`title_id=0xFFFE07D1`** (the SYSTEM "profile" title,
not the game) — 3 settings from the id-table at `0x82610C24` into `[r31+160]`. That's the system
gamer-prefs path (the `0x1004xxxx` settings), **not** the campaign progress. So it's not the
g_slots loader. ⇒ The campaign load really does go to `0x828E3A38` while the save serializes g_slots
`0x828EB348` — the asymmetry is genuine, not a second path I missed. **Crux for the fix:** the
relationship between `0x828E3A38` (load target) and `0x828EB348`/g_slots (save source + what the
level-select reads). Likely `0x828E3A38` is a save-handle/staging table and the
`0x828E3A38 → g_slots` apply is missing or not run on CAMPAIGN entry. (The addresses are immediate
lis/addi, so not a recomp relocation bug.) Next: dump/inspect the `0x828E3A38` struct the load fills
(`[slot+52]` from `sub_822A2150`) and find who, if anyone, copies it into g_slots.

## Layer findings (2026-05-24, deeper trace)
- **LOAD mgr `sub_8229C8D0` is GENERIC** — many callers (recomp.0/.2/.3, 9+ sites). It's a
  "load setting blob into the `0x828E3A38` per-(user,setting) cache" helper, not campaign-specific.
  So `0x828E3A38` is a generic settings cache; the campaign-specific "apply cache → g_slots" is
  elsewhere (game logic).
- **SAVE trigger = `sub_82150770`** (recomp.2:38219) → `sub_82296A38` (save mgr) → serializes
  g_slots (`0x828EB348`). `sub_82150770` is in the **0x8215xxxx save subsystem** (cf. the dirty-flag
  pump `sub_82151170` / unconditional `sub_8215E2E0` / handler `sub_8215D348` in [[55-save-system]]).
  So the save just *persists* g_slots; it is NOT the reset.
- ⇒ **The two targets for the fix:** (a) the **reset** that defaults g_slots on LOCAL-GAME/lobby
  entry (game-specific, many layers up — likely near the session-enroll/lobby-init `sub_82297*`/
  `sub_82298*` code; note the session-enroll fix routed the signed-in player through `loc_82298008`
  — check it doesn't bypass a profile-apply), and (b) the **missing apply** that should copy the
  loaded `0x828E3A38` cache into g_slots on CAMPAIGN entry. Both are guest-side.

## Enroll/slot-init checked — NOT the apply point
`sub_82297F30` `loc_82298008` (the session-enroll fix path) inits only the **session-player** part
of the g_slots slot: `r3=r31+2488; bctrl` (init the +2488 obj), `stw r11,2388(r31)` (set state).
It does **not** call the load mgr and does **not** populate the campaign-progress fields. So the
campaign-progress **apply into the slot is a separate step on CAMPAIGN entry** (after the lobby) —
that is the missing/broken piece (fix target b). Trace the CAMPAIGN-mode-select handler: it should
copy the loaded `0x828E3A38` cache (or re-read the profile) into the g_slots slot's progress fields;
if it doesn't (or a mis-translated fn), the slot stays default → level-select resets.

## PRECISE mechanism (deepest trace) + the exact fix candidates
- `sub_8229C8D0(key=XUID)` (load mgr) loads the `0x63E83FF*` blobs into the per-user cache slot
  (`0x828E3A38`, found by `sub_8229CA28`) and **returns that slot** (ends `mr r3,r29; return`).
- It has **9 callers** that consume the loaded slot (named):
  `sub_82103980` (recomp.0:8907), `sub_821046A0` (0:10871), `sub_821049C0` (0:11346),
  `sub_82154B08` (recomp.2:48640), `sub_82160EF0` (2:77688), `sub_821648F8` (recomp.3:8084),
  `sub_82167890` (3:14734), `sub_82168748` (3:16901), `sub_82169D88` (3:20168). The CAMPAIGN-entry
  apply is one of these (the recomp.3 `sub_8216*` cluster is the likely UI/campaign area). For each,
  check: does it write g_slots (`0x828EB348`) from the returned `r3` slot? The one that *should* but
  doesn't (or is mis-translated) is the fix site.
- The **level-select unlock reads g_slots** (`0x828EB348`), NOT the returned cache slot — proven:
  the extended guard preserved ALL 3 disk blobs across restart (`FFE=2`,`FFF=22` non-zero) yet the
  level-select still showed only Stan's House. So the **loaded `0x828E3A38` slot is never copied
  into the g_slots progress fields**; g_slots is re-init'd to default on LOCAL-GAME entry.
- **XUID is a stable constant** (`0xB13EBABEBABEBABE`, user_profile.cpp), so it is NOT a key
  mismatch — the load and any apply hit the same cache slot.
- ⇒ **The fix:** one of the 9 `sub_8229C8D0` callers (the CAMPAIGN-entry path, most likely in the
  menu/campaign code = recomp.2/.3 sites) must copy the returned slot's loaded progress into the
  g_slots session-player slot's progress fields — that copy is missing or mis-translated. Examine
  those 9 callers (what they do with the returned `r3` slot; which writes g_slots `0x828EB348`),
  find the CAMPAIGN-entry one, and restore the copy via a config function-override / fixup.
  **Fastest localize:** instrument the generated `sub_8229C8D0` (log each call + caller LR + whether
  the returned slot holds progress) and run boot→CAMPAIGN→level-select; the call on CAMPAIGN entry
  whose result is dropped is the bug site. (App rebuild needed for generated-code instrumentation.)

## Final model (this session's deepest finding)
`sub_821648F8` (one of the 4 g_slots-region candidates) is an **on-demand QUERY**, not a copy: it
`sub_8229C8D0`-loads the user's campaign blob, then returns a single field (`[slot+20]`, gated by
`[slot+68]==0`) to its caller's output — it does NOT write g_slots. So the loaded profile data is
**queryable on demand** (the game CAN read saved progress via these helpers), yet the **level-select
reads g_slots (`0x828EB348`) directly**, and g_slots is only ever populated by the **in-session
level-complete logic**, never from the profile on a fresh boot. ⇒ The true gap is **"g_slots is not
seeded from the saved profile on sign-in / CAMPAIGN entry"** (or the level-select should use the
on-demand query but uses g_slots). Remaining candidates to check for a g_slots *write* from the
load: `sub_821046A0`, `sub_82167890`, `sub_82169D88` (the other 3 that touch the g_slots region).
If none writes g_slots, the fix is to add the seed (config function-override that, on CAMPAIGN
entry, copies the on-demand-queried progress into g_slots) — guest-side, app rebuild + play/restart
verify. **This is multi-session; the mechanism + candidates are now fully pinned for resumption.**

## All 4 candidates are the SAME on-demand query, gated by `[slot+68]`
`sub_821046A0`, `sub_821648F8`, `sub_82167890`, `sub_82169D88` all do: `sub_8229C8D0`-load →
`if r3==0 → 0; r11=[r3+68]; if r11!=0 → 0; else r11=[r3+20]` → store to the caller's OUTPUT (not
g_slots). So they're **on-demand reads of the loaded campaign field `[slot+20]`, GATED by
`[slot+68]==0`**. None seeds g_slots. Crucially, **`[slot+68]` is written by the load itself** —
`sub_8229CB38` line ~8568: `[slot+68] = !(r30 & 1)` where `r30` is the load result/error. So **if
the load's result makes `[slot+68]≠0`, every query returns default (0)** regardless of the saved
bytes — which would explain "saved on disk, preserved by the guard, but the game still reads
default." 

### THE next step (needs runtime instrumentation — app rebuild)
Instrument the load: in `sub_8229CB38`/`sub_8229C8D0` (or via the SDK read result), log the slot's
`r30` result and the resulting `[slot+68]` and `[slot+20]` for the campaign blob, during
boot→CAMPAIGN→level-select. If `[slot+68]≠0`, the load is being treated as "invalid/no-save" — fix
the load-success path (why `r30` indicates failure for a valid profile in the stub/guest config).
If `[slot+68]==0` but the level-select still shows default, the level-select doesn't use these
queries (reads g_slots) → seed g_slots. Either way it's guest-side + verify-by-running, multi-session.

## How to make progress (concrete)
- **Instrument the save-slot global:** log reads/writes of `0x828E3A38` region (or the
  `sub_8229CA28` return) with the key, at boot vs at the level-select, to see if the same slot is
  used and whether the unlock data survives into the level-select read.
- **Find the level-select unlock source:** trace the CAMPAIGN-LEVEL-SELECT screen's "is level N
  unlocked" check (it ran when the grid drew showing Elementary School unlocked *in-session* after
  the Stan's House win — that write path is the in-memory progress; find the global it set).
- **Find the reset:** what runs on LOCAL-GAME/lobby entry that defaults the campaign progress
  (the clobber-save `sub_82296A38` serializes whatever that left). A guest backtrace at
  `sub_82296A38` (or `0x82298418`) during nav, symbolized, gives the caller chain.
- The runtime has `[PROF-*]` diagnostics (in `patches/rexglue-sdk-current-full.patch`); add
  `0x828E3A38`-watch logging similarly.

## Why no runtime-only fix
Verified a `UserProfile::SaveSetting` guard (refuse a write with fewer non-zero bytes than disk for
`0x63E83FF*`) **preserves the disk save across restart+nav** — but the level-select still reset. So
the game ignores the (preserved) disk for the unlock display; the fix must make the **guest** apply
the loaded `0x828E3A38` slot to the campaign-progress state. That's guest-side: a config function
override / post-codegen fixup, or finding+correcting a mis-translated function in the apply path.
Guard reverted (kept the runtime clean); reinstate it only alongside a working guest-side apply.
