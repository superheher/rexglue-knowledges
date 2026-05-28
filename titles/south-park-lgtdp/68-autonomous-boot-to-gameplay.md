# Autonomous boot → live gameplay in one command (2026-05-28)

`tools/gamectl.sh play` takes a cold shell to **live tower-defense gameplay on the first
level (Stan's House), un-paused, in ~60–70 s, fully hands-off** — no screenshot babysitting,
no fixed sleeps to guess at. This documents the recipe and the non-obvious behaviours it has
to work around, all **verified by running** (Fedora 43, RADV, warm build with shaders cached).
Companion to [[50-menu-input-and-lobby]], [[60-boot-present-deadlock]], [[70-video-playback]],
[[67-polish-backlog]].

## TL;DR recipe

1. **Kill** every `south_park_td` (exact-name match) — only one instance may run.
2. **Launch** one fresh, detached instance; **wait for the first `[pacing-diag]`** line
   (≈ first present). No line within 15 s ⇒ boot hung ⇒ relaunch (retry ≤ 4×).
3. Press **START** once to leave the title.
4. **Alternate A / START** every ~1.8 s until **`camp_diagram` appears in the log**
   (= CAMPAIGN LEVEL SELECT is on screen — a reliable checkpoint).
5. Press **only A** ~13× to select Stan's House → confirm → start → skip the "LEVEL 1"
   card and the in-level dialogue → **live gameplay**.

`tools/gamectl.sh bench [sec]` then samples `swaps/s` from `[pacing-diag]` (min/avg/max).

## Why each step is the way it is

### Warm boot is ~4 s, but boots intermittently HANG
With shaders cached (`Created 41 graphics pipelines from Vulkan storage`) the title renders
~4 s after launch — the first `[pacing-diag] swaps …/s` line marks it. (Cold first boot is
~1–3 min of shader translation; the old "~55 s" in [[50-menu-input-and-lobby]] was cold.)
But some boots **dead-end before the first present** (run.log stops at the `PROF-RD` lines,
**zero** `pacing-diag`) — the frame-1 vsync/fence deadlock of [[60-boot-present-deadlock]],
still intermittently reachable. Detection is therefore "no `pacing-diag` in 15 s → relaunch",
which is what makes `play` reliable despite it.

### The title takes only START, and only while FRESH
"PRESS START" ignores A entirely (mashing A there does nothing). And input is reliable **only
on a fresh process**: once an instance has idled ~28 s it plays its **attract demo**
(`towerDefense_attract_movie.wmv` — the "BUILD SNOW MAZES…" feature cards, see
[[70-video-playback]]) and goes effectively **input-dead**, looping title→attract→title. So
`play` always drives a just-launched instance. Corollary: **never run two instances** — they
share one `live_input.txt`, so inputs hit both and the windows desync.

> Footgun: kill with `pkill -x south_park_td` (exact comm). `pkill -f south_park_td` also
> matches the harness's own shell (its args contain the string) and kills it mid-run.

### A ~30 s unskippable intro sits between START and the menu
After START: Comedy Central → doublesix → legal text → in-engine opening cutscene (~30 s),
*then* the menu. There is **no skip cvar** (only `skip_arcade_logo`, already on). Pressing
A/START during it does not shortcut it — the ~30 s is the floor on `play`'s wall-clock.

### `loading=true` is NOT a usable "entered the level" signal
The `[pacing-diag] … loading=<bool>` flag is computed from `LastPipelineHostTick` — i.e. it is
true only when a **shader is compiling**. On a warm machine the first level's shaders are all
cached, so it loads with **`loading=false` throughout** (observed: 0 `loading=true` across a
full run that did reach the level). The level also loads **silently** — no distinct log line.
So `play` keys off the `camp_diagram` checkpoint for the menu and the visible HUD / "LEVEL 1
STAN'S HOUSE" card for entry, not the loading flag.

### Why alternate A/START, then switch to pure A
Title→campaign-select needs both buttons (START for the title/lobby gates, A for the
mode/game menus) but their order shifts with intro-timing jitter, so a fixed sequence is
fragile — blind alternation covers it. **After** `camp_diagram`, the path to gameplay is
all-A (select → confirm → start → skip dialogue); pressing START there would pause or
back out. Ending on pure A also guarantees we land **un-paused** (a stray START as the level
goes live leaves it on GAME PAUSED). Extra A's in-match are harmless — they just select/place,
and `--always_win` keeps the base invincible.

### Performance signal for profiling
In-gameplay framerate **oscillates ~25–60 swaps/s** on Stan's House (RADV). The level-entry /
intro-dialogue phase sits ~25–35; steady play touches 60 and dips under GPU load. This is the
genuine drop to profile — distinct from the old sim-speed wobble, since the current
`command_processor.cpp` frame limiter (see below) caps gameplay at refresh and skips throttling
during loads.

## Runtime/.so state this was verified against
The `librexruntime.so` in use carries an **uncommitted, evolved** `command_processor.cpp`
pacing change (beyond the working-tree patch): a frame limiter in `ExecutePacketType3_XE_SWAP`
that throttles to refresh **only during gameplay, not loading** + `RefreshVblankFence()` +
periodic ring read-pointer write-back (boot-intro deadlock mitigations) + a WAIT_REG_MEM
500k-spin escape + `clear_memory_page_state` defaulted OFF (glyph fix, [[65-font-glyph-corruption]]).
This is the "distinguish loading from gameplay" approach [[60-boot-present-deadlock]] called for;
it is not yet folded into `patches/`, so a fresh `build-linux.sh` will behave differently until
that patch is regenerated.
