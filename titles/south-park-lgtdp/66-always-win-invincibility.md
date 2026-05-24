# `--always_win` invincibility cheat — can't lose a level (2026-05-24)

**Status: ✅ DONE — implemented + verified by running.** A new runtime cvar
`--always_win=1` makes the base (school) invincible so the player can't get a GAME OVER.
Implemented entirely runtime-side (no game-asset edits), as a patch to the rexglue SDK.

## What the user asked for
"A flag for infinite lives / invincibility, so you can't lose levels and there's always a
victory ahead." The user chose the approach **"block the GAME OVER trigger."**

## The key insight (and the wrong turn that running caught)
The game's lose/win **outcome is decided by the engine (the recompiled guest code), not by
Lua.** When the base's defense health reaches 0 the engine records a defeat (reason
`DefenseKilled`) and shows its own **GAME OVER** menu (TRY AGAIN / RESTART LEVEL / ...). The
engine then *also* calls the Lua `EndGame(reason)`, which dispatches to
`GenericLoseLevelSequence` (`media/Assets/LuaScripts/LevelGlobalFunctions.lua:383-397`) — but
that Lua function is **only presentation** (it sets "Lose" music, waits, calls
`SignalEndLevel`). It does **not** decide the outcome.

**First (wrong) approach:** redefine `GenericLoseLevelSequence` to run
`GenericWinLevelSequence` instead, so "every loss becomes a win." A *forced* Lua call
`EndGame("DefenseKilled")` did show **STAGE COMPLETE** — which looked like success — but that
was **misleading**: forcing `EndGame` from Lua means the engine never recorded a loss, so it
showed the result the win-sequence implied. When a **real** loss was triggered (zero the base
health so the *engine* detects it), the screen showed **GAME OVER** anyway — the Lua
redefinition had no effect on the engine's recorded result. This was only caught by **running a
real loss**, not the forced one. Lesson promoted to `general/95-pitfalls-and-patterns.md`.

**Correct approach — prevent the lose condition.** The only reliable way to block the GAME OVER
is to stop the engine from ever seeing defense health <= 0, i.e. make the base unkillable.

## Implementation
`SetDefenseHealth(n)` / `GetDefenseHealth()` are Lua-callable (used by the difficulty setup,
e.g. `SetDifficultyCasual` → `SetDefenseHealth(100)`, and `StanTriggerSpecial`). The cheat
appends a small keep-alive to `LevelGlobalFunctions.lua`: it wraps `LaunchStage` (the per-stage
entry the engine calls) to start one background `Task` that, every 0.1 s, restores the defense
health whenever it has dropped below the observed max:

```lua
local __aw_LaunchStage = LaunchStage
function LaunchStage(s,p,c,d)
  __aw_maxhp = 0
  if not __aw_keepalive then
    __aw_keepalive = true
    Task(function()
      while true do
        local ok, h = pcall(GetDefenseHealth)
        if ok and h then
          if h > __aw_maxhp then __aw_maxhp = h end
          if h > 0 and h < __aw_maxhp then pcall(SetDefenseHealth, __aw_maxhp) end
        end
        Wait(0.1)
      end
    end)
  end
  __aw_LaunchStage(s,p,c,d)
end
```

Delivery is the same redirect trick as nothing-else-touches-the-assets: `HostPathEntry::Create`
(`src/filesystem/devices/host_path_entry.cpp`), when `REXCVAR_GET(always_win)` is set and the
file is `LevelGlobalFunctions.lua`, writes `<temp>/rexglue_alwayswin_lgf.lua` = original +
keep-alive and points the entry's `host_path_` at it (works for both `ReadSync` and mmap). Boot
logs `[always_win] redirected LevelGlobalFunctions.lua -> ... ; the base is now invincible`.
Kept as a patch in `south-park-recomp/patches/rexglue-sdk-current-full.patch` (not committed to
the submodule, per project convention).

## Verification (by running)
A temporary debug cvar `always_win_selftest` (since removed) added a *drain* task that did
`SetDefenseHealth(GetDefenseHealth()-5)` every 0.05 s ≈ **-100 HP/s** — which on its own zeroes
a 100-HP base in ~1 s → GAME OVER (confirmed earlier with `SetDefenseHealth(0)`). With
`--always_win=1` **and** the drain both running, ELEMENTARY SCHOOL (Level 2, CASUAL) ran for
**40 s+ with enemies piled up at the base and NO GAME OVER** — the base health bar stayed green
(screenshots `C:\Temp\r7_drain1/2.png`, `r7_proof.png`). Because the drain is far harsher than
natural enemies and the keep-alive is exactly what shipping `--always_win=1` runs, the shipping
config is verified a fortiori. The self-test was then removed and the final build re-confirmed
to boot and redirect correctly.

## Scope / limitations
- Covers the primary lose condition **`DefenseKilled`** (base destroyed) — the "school under
  attack" defeat. The other engine lose reasons (`AllCharactersKilled`, `TimeUp`) are not
  specifically handled; if a mode relies on them, they'd need their own keep-alive (keep
  characters alive / extend the timer). For this title's campaign defense levels, `DefenseKilled`
  is the relevant one.
- You still have to play (clear waves) to *win*; the cheat only removes the ability to *lose*.
- The keep-alive `Task` runs for the session (one instance); in menus `GetDefenseHealth` is
  `pcall`-guarded so it no-ops.

## Files
- `third_party/rexglue-sdk/src/filesystem/devices/host_path_entry.cpp` — cvar + redirect + keep-alive.
- `south-park-recomp/patches/rexglue-sdk-current-full.patch` — the SDK change.
- `south-park-recomp/docs/RUN.md` — user-facing flag doc.
