# 67 — Polish backlog (post-v1): assets, audio, boot, intro movie

Status of the post-v1 polish items (v1 = playable boot→menu→match→win + save +
cross-restart continue + in-match font fix + `--always_win`). Each item below was
**verified by running** (build: rexglue-sdk RelWithDebInfo, `--license_mask=1`,
focus-free input). Companion to docs 55/56/60/65/66 and `general/95`.

---

## 1. Elementary asset gap (`en-en` locale subdir) — ✅ SOLVED (runtime fallback)

**Symptom.** At the CAMPAIGN LEVEL SELECT the log showed `NtCreateFile FAILED ... 0xc000000f`
for `game:\media\Assets\Frontend\Graphics\en-en\{camp_diagram_*,local_campaign_slide_*}.png`
(the per-level map diagrams / slides) — and `LuaScripts\School.lua`.

**Root cause.** The retail disc lays *localized* assets under a locale subdirectory
(`...\Frontend\Graphics\en-en\foo.png`, just like `...\Movies\en-en\bar.wmv`), but the flat
asset extraction dropped that subdir — the files live in the parent dir
(`camp_diagram_school.png` exists at `...\Graphics\`, not `...\Graphics\en-en\`). This is the
**same class** as the earlier movie `en-en` issue (previously worked around by manually
hardlinking the WMVs into a `Movies\en-en\` dir).

**Fix (runtime, general).** `NtCreateFile_entry` (`xboxkrnl_io.cpp`): when a **pure open**
(`FileDisposition::kOpen`) fails and the path contains a **locale tag** — a middle path
component matching `xx-yy` (two ASCII letters / `-` / two ASCII letters) — retry **once** with
that component removed, serving the base asset from the parent dir. Helper
`TryStripLocaleSegment`. Logged as `[NtCreateFile] locale-fallback: served '<parent>' for
'<locale path>'`. It only fires on failure, so a genuinely-localized file (if present) still
wins; the retry is harmless if the parent file is also absent. **This makes the manual movie
`en-en` hardlink step unnecessary and is title-agnostic.**

**Verified by running.** Navigated boot→…→CAMPAIGN LEVEL SELECT: **8 `locale-fallback: served`
hits** (`camp_diagram_Stans_house/school/generic`, `local_campaign_slide_school`), **0**
remaining `en-en` `0xc000000f` failures, and the **Elementary School map diagram renders** on
the level-select screen (screenshot `C:\Temp\sp_n5b_levelselect.png`).

**`School.lua` — won't-fix (benign).** `game:\media\Assets\LuaScripts\School.lua` is **not in
the dump** (no `School.lua`/`school.lua` anywhere; the locale fallback doesn't apply — it's not
a locale path). The `LuaScripts` dir has only shared scripts (`LevelGlobalFunctions`,
`MissionGlobal`, `GlobalScripts`, `InitTutorial`) + per-character/-enemy scripts — **no
per-level script** named after any level. `School.lua` is an **optional per-level hook script**
the engine probes for and tolerates when absent. **Verified: Elementary School plays fully
without it** (screenshot `C:\Temp\sp_n7_match2.png` — schoolyard map, units, HUD, waves). Left
as-is.

---

## 2. Audio fidelity — ✅ INVESTIGATED + clips produced; NEEDS A HUMAN EAR (can't be agent-signed)

The XMA→SDL path is **objectively correct** (re-confirmed by reading the code + capturing the
output); the only open question is subjective fidelity, which requires listening.

- **Downmix is correct.** Device opened **stereo** (2ch), so the runtime folds the guest's 6ch
  (5.1) down via `sequential_6_BE_to_interleaved_2_LE` (`conversion.h`, AMD64 SIMD path):
  `L=(FL+BL+0.5·FC)·(1/2.5)`, `R=(FR+BR+0.5·FC)·(1/2.5)`, LFE dropped, BE→LE byte-swap. This is
  the standard XAudio2 5.1→stereo fold (center −6 dB to both, surrounds summed, normalized to
  avoid clipping). Channel indices on the SIMD path match XAudio2 (FL,FR,FC,LFE,BL,BR). *(Note:
  the scalar `#else` fallback has BL/BR swapped, but it is not used on AMD64.)*
- **Capture tool added** (`audio_dump` cvar, `sdl_audio_driver.cpp`): `--audio_dump=<file>`
  appends the exact interleaved F32LE PCM handed to SDL. Convert:
  `ffmpeg -f f32le -ar 48000 -ac <channels> -i <file> out.wav`.
- **The captured audio is real-time and clean.** A full menu→match capture: **peak 0.43 (no
  clipping)**; the **real (non-silence) audio = 235.9 s ≈ the 264 s wall-clock** → the game
  produces audio at **real-time (1×)**.
- **⚠️ Capture-environment artifact (NOT a port bug): SDL drained the stream at exactly 3×
  real-time** on this headless/remote box, padding the dump with **70.4 % exact-silence
  frames** (the `SDLCallback` "no frames queued → memset 0" branch). This happens when the
  audio endpoint does not pace playback (a virtual/non-rendering sink — this machine has
  SteelSeries Sonar virtual devices + monitor HDMI endpoints). On a normal output device SDL
  blocks at real-time and this does not occur. The **samples are correct**; only the *cadence*
  of this capture is inflated. To recover a real-time clip, strip the exact-zero frames
  (`C:\Temp\strip_audio.py`).
- **Deliverables for the maintainer** (lossless, so fidelity isn't tainted by codec):
  `C:\Temp\sp_audio_full.flac` (~236 s — boot/menu/match) and `C:\Temp\sp_audio_match.flac`
  (~53 s in-match). **The maintainer (real audio device) is the sign-off** — listen for
  distortion / wrong pitch / dropouts. An agent cannot ear-verify.

---

## 3. Boot speed — ✅ PROFILED; the "4–5 min" figure was stale (cold shader cache)

Measured boot-to-title on a warm cache by timed screenshots:

| t (s) | screen |
|---|---|
| 0–1 | runtime init complete (`Runtime initialized successfully`, <1 s) |
| ~15 | publisher splash ("SOUTH PARK DIGITAL STUDIOS") |
| ~45 | animated in-engine intro (town backdrop + "Ⓐ SKIP") |
| **~50–60** | **TITLE SCREEN ("PRESS START")** |

**Boot-to-title ≈ 50–60 s warm**, not 4–5 minutes. Findings:
- The **CP `WAIT_REG_MEM` fence loop is healthy**: **0** "stuck" detections this run (fences
  resolve within the 8000-spin fast-poll budget added in doc 60). It is **not** the bottleneck.
- Runtime/CRT init is **<1 s**; shaders/pipelines load **from the warm cache** ("Translated 28
  shaders **from the storage** in 1 ms"). The earlier ~4–5 min was almost certainly a **cold
  cache** first boot (one-time live shader translation of the whole set) — inherent and cached
  thereafter — and/or the old pre-spin-yield build.
- The remaining ~50–60 s is **game-paced splash + intro animation**, which is **user-skippable**
  ("Ⓐ SKIP"); it is game logic, not a runtime stall. **No cheap runtime speed-up remains.**

Conclusion: no action needed beyond the doc-60 spin-yield. (A cold-cache first boot is slower;
that is one-time shader translation, not a bug.)

---

## 4. Intro / cutscene movies (WMV) — ✅ ACTUALLY WORK (the "black & silent" note was outdated)

**CORRECTION (2026-05-24, verified by running; full detail: doc `70-video-playback.md`).** The
movies are **NOT black & silent** in the current build — the title's **own in-software WMV3 (VC-1)
video + WMA2 audio decoders render them** (video *and* audio). This earlier "no decoder → black"
note was an **assumption that was never screenshot-verified**.

- **Format (ffprobe):** `sp_xbox_0_intro.wmv` + `LevelN{Intro,Mid,End}.wmv` = WMV3 (VC-1) video,
  1280×720, 24 fps + WMA2 audio, 44.1 kHz stereo, ASF, ~22 s. (Same format ⇒ same code path.)
- **What's true:** the title imports **no** system video API; it **decodes the .wmv in-guest**
  (reads the whole file in one `NtReadFile`, decodes from memory; the movie is a per-frame **scene
  node**). Verified by running: the intro plays **animating video + audio** then completes/advances;
  removing the `.wmv` → the scene goes **black** (so the *game* decodes it); an audio capture shows a
  ~26 s music segment matching the WMV's audio track.
- **Why it was black before (best theory):** the GPU shared-memory **page-validity bug (doc 65)** —
  fixed this same day — corrupted/blocked the decoded movie frame's texture upload; the in-software
  decoders were always present. Once doc 65 landed, the movie video appears.
- **Mitigation still in place:** user-skippable ("Ⓐ SKIP", the game's own prompt); boot doesn't
  block on it.
- **Don't add a host decoder:** a full cross-platform libavcodec VC-1/WMV3 + WMA2 path was built +
  proven during the investigation, then **reverted** (maintainer scope decision) because the guest
  already plays the movies. Recipe to restore it is in doc 70 §4 if the guest path ever regresses.

---

## 5. Online features (co-op / leaderboards / achievements / avatars) — OUT OF SCOPE for v1

Left **stubbed offline** (the XAM party/session/live imports resolve to stubs; the title runs
as a single offline user with one storage device). v1 is the **offline single-player** loop.
Not pursued.

---

## 6. Window UX — ✅ cursor freed, window icon, blocky boot logo skipped (maintainer-requested)

All three are SDK/DLL-level (no exe rebuild), cvar-gated, verified by running.

- **Cursor was captured/locked to the window.** The mnk driver's `UpdateMouseCapture` hid the
  cursor + `CaptureMouse()` + re-centered it every frame (for mouse-look) whenever mnk was enabled
  and the window focused → the cursor couldn't leave the window, making it impossible to move/resize.
  Fix: **`mnk_mouse_look` cvar, default OFF** — `should_capture` is gated on it, so the cursor stays
  free + visible. Mouse **buttons still map to triggers** (read from key state, independent of
  capture); only mouse-movement→right-stick is lost (opt back in with `--mnk_mouse_look=1`).
- **No window/taskbar icon** (`LoadIconW(hinstance,"MAINICON")` is null — the exe has no icon
  resource). Fix: **`window_icon` cvar** = path to a `.ico`; `window_win.cpp` loads it via
  `LoadImageW(LR_LOADFROMFILE)` and sets it for `ICON_BIG`/`ICON_SMALL` + the window class. The 64×64
  `SouthPark.png` → a multi-size `SouthPark.ico` (16/32/48/64, made with Pillow); launcher passes
  `--window_icon`.
- **The first boot image (Xbox Live Arcade logo) is blocky** — it's `media/ArcadeLogo.ptc`, a
  `PTC+MSHM` multi-frame container whose frames are **JPEG-compressed** (found `FF D8 FF` streams
  inside), so the full-screen splash shows 8×8 DCT block + ringing artifacts; the rest of the UI is
  lossless PNG/TGA → crisp. It's the **source asset**, not a decode bug (the runtime decodes the JPEG
  faithfully — same JPEG path as the boot fix). The artifacts are baked into the lossy data; can't be
  un-compressed. Maintainer preferred to hide it → **`skip_arcade_logo` cvar**: `NtCreateFile` reports
  `ArcadeLogo.ptc` not-found, so the game skips it and boots straight to the (crisp) Microsoft Game
  Studios splash. Verified: logo gone, **no stall** on the missing asset, boot reaches the title.
  (Lesson: a "first image blocky, everything after crisp" split usually = that one asset is lossily
  compressed in the source; check its magic bytes — JPEG/DXT — before suspecting the decoder.)
