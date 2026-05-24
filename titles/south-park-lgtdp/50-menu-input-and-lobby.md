# Menu, input, and the lobby — reaching an interactive game (2026-05-24)

After the image-load SEH fix (see [[40-seh-implementation-plan]]) the recomp goes
**boot → intro → title → MAIN MENU → LOCAL GAME → LOBBY**, all rendering correctly and
**responding to input**. This logs that journey, the input technique, and the next blocker.

## The navigable flow (screenshot-verified)
| Screen | Reached by | Notes |
|---|---|---|
| Title ("PRESS START") | boot (~55–60 s) | the four boys + logo |
| Main menu | **Start** | UNLOCK FULL GAME / **LOCAL GAME** / XBOX LIVE GAME / SCRAPBOOK / LEADERBOARDS / ACHIEVEMENTS / HELP & OPTIONS / EXIT GAME |
| Lobby | **A** (selects LOCAL GAME) | "1/4 SIGNED IN", USER + OPEN slots, "Ⓑ BACK" |

The guest polls `XamInputGetState` (it queries **both user 0 and user 2**) every frame and
**responds** — so the input path is functionally correct end to end.

## Input: synthetic OS injection does NOT work here — use the mnk injector
- `keybd_event` / `SendInput` / `PostMessage` **never reach SDL** in this environment, even
  with the game as `GetForegroundWindow()` and after focus transitions (minimize→restore,
  click, AttachThreadInput). A temporary log in `MnkInputDriver::OnKeyDown` **never fired** →
  the synthetic keys don't reach SDL's keyboard layer at all (UIPI/session integrity, or SDL
  not having keyboard-focus; other "PS2-Recomp" windows on the desktop also steal foreground).
  This is an **environment/automation limitation, not a port bug**.
- **Solution (committed, env-gated):** an injector in `MnkInputDriver::GetState`
  (`REX_INJECT_SCRIPT="t:hexbtn,…"`, `REX_INJECT_DUR`) presses XInput buttons at time `t`,
  **bypassing the focus gate**; it OR's into the merged XInput state (`InputSystem::GetState`
  merges all drivers) and edge-enqueues keystrokes for `GetKeystroke`. Button masks:
  `0010`=START, `1000`=A, `2000`=B, `0001/2/4/8`=DPAD. E.g. `REX_INJECT_SCRIPT="66:0010"`.
- **A real user** with a focused gamepad/keyboard should navigate normally (the plumbing —
  `XamInputGetState`/`GetKeystroke` ← `input_system` ← mnk/SDL, mnk `has_focus_` default true —
  is verified correct). General lesson promoted to `general/95`.
- **Don't rapid-repeat Start at the title** — re-entrancy hits the crash below; single presses
  with gaps are stable.

## Open blocker #1 — lobby → match crash (the gate to gameplay)
Pressing A or Start **in the lobby** (begin / ready-up) faults:
`SEH 0xC0000005 fault_addr=0x1A in sub_82101AF0+0xF9` (bt …`sub_82156038→sub_82101C08→sub_82104FD0`).
Root cause (read the codegen):
- `sub_82101AF0(playerIndex)` = "is player ready?" — guards on the player being **active**
  (bit `playerIndex` in `[g_base+1600]`, `g_base ≈ 0x828EB348`), then `bl sub_82297C48; r10=[r3+26]`
  — **derefs the result unconditionally**.
- `sub_82297C48(i)` returns `&g_slots[i*3176]+2488` **only if the slot state `[slot+2388] ∈ {3,4}`**
  ("ready"), else **null** → the `[null+0x1A]` fault.
- So the slot is **active but its state ≠ 3/4** — the player was never transitioned to "ready".
  The ready setter exists (recomp.12.cpp ~line 74789: `…bctrl; li r11,3; stw r11,2388(slot)`),
  and the reset setter (74130) writes 0. The active USER slot isn't reaching state 3.
- **Next:** trace who calls the ready-setter + the lobby input→action dispatch (is the A press
  meant to call it? does its `bctrl` fault? is a `xam` sign-in/profile stub leaving the slot
  half-initialized?). Forcing `sub_82297C48` non-null only defers it — the ready FLAG
  `[slot+26]&1` must also be set. A lobby/sign-in **state-machine** problem, not a 1-line bug.

### UPDATE (root-caused) — it's a CLASS, and the local player has no session object
The lobby→match crash is **one of a class**: many functions call `sub_82297C48(playerIndex)`
to get a **session-player object** (`&g_slots[i]+2488`) and **deref it unconditionally**
(`sub_82101AF0` at `[r3+26]`, `sub_8216A8E8` at `[r3+28]`, … the whole `sub_8216Axxx` family).
`sub_82297C48` returns that object **only if slot state `[slot+2388] ∈ {3,4}`** ("session
player"), else null. Live data: the **local signed-in player is slot 0 with state = 1** (set by
`sub_82297F30`'s "signed-in" else-path; empty slots 1–3 get state 3 via its other path, which
also runs a `bctrl` that **initializes `+2488`**). So the local player is **active but is not a
session player**: state 1, and its `+2488` object was never initialized. Every session-player
query on it returns null → crash.
- **✅ ROOT-FIXED → reaches an in-game MATCH.** In `sub_82297F30`, drop the local player's
  state-1 shortcut (`li r11,1; goto loc_8229802C`) so its else-path **falls through into the
  session-enroll path `loc_82298008`** (which runs the `bctrl` that inits `+2488` and sets state
  3) — i.e. enroll the local signed-in player as a session player. Persisted as
  `fix_recomp_labels.py` **Fix 6** (post-codegen; the generated file is git-ignored). **Verified
  on a clean regen** (no diagnostics/guards): slot-0 state becomes **3**, the whole crash class
  is gone, and navigation reaches the actual tower-defense **MATCH** — `boot → intro → title →
  main menu → LOCAL GAME → lobby → game mode (Campaign) → level select (Stan's House) → MATCH`
  (snowy map, enemy path, character units; screenshot-verified). The earlier `sub_82101AF0`
  null→ready guard is now unnecessary. **Remaining for full playability:** play through to
  win/lose, save/continue, audio (XMA→SDL), and the non-deterministic GPU-fence stall.

## Open blocker #2 — non-deterministic GPU-fence stall (pre-input on some runs)
The main thread sometimes spins in `sub_821C6E58` (`while (*[obj+10896] < target) { if
(!sub_821B9270()) break; }`) waiting for a **guest GPU fence** that doesn't advance → that run
never reaches input (0 `XamInputGetState` calls). Usually the fence advances and input is
polled normally. Proper fix = **runtime GPU fence write-back** (the CP must write the fence
value to guest memory on completion); a per-call iteration cap in the spin is a stop-gap.

## Status vs. Condition A
Interactive **to the lobby** — but **not** through a match → win/lose → save. The two blockers
above (lobby-ready state machine, GPU fence), then the match gameplay itself, are the remaining
multi-session work. Everything to the lobby is committed and reproducible (config-only fixes +
the env-gated injector in `patches/rexglue-sdk-current-full.patch`).
