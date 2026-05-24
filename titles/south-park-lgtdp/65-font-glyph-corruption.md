# Rendering defect — some font glyphs render as striped/garbled boxes (2026-05-24)

**Status: ✅ SOLVED 2026-05-24 — fixed + verified by running (clean A/B).** It was NOT a
tile-mode/pitch/format bug — it was a **cross-thread data race in the GPU shared-memory page-validity
tracking**. See the SOLUTION section below; the original symptom notes follow it.

## ✅ SOLUTION (2026-05-24)

**Root cause — a stale-data race, NOT a tiling/decode bug.** rexglue rewrote Xenia's
`SharedMemory` valid-page tracking into a **lock-free double-buffered flag scheme**
(`active_valid_flags_` / `staging_valid_flags_`). The frame-end buffer swap
`SetSystemPageBlocksValidWithGpuDataWritten()` (`src/graphics/shared_memory.cpp:126`) ran the
`active_valid_flags_.exchange(staging)` **WITHOUT** the global lock, while
`MemoryInvalidationCallback()` (`:587`, runs under the lock on guest CPU writes) loads the active
pointer then clears the invalidated page bits through it. Interleaved: a guest thread loads the
active ptr → the GPU thread swaps → the guest clears bits in the now-**retired** buffer → the
just-invalidated page stays VALID in the new active buffer for that frame → the `RequestRanges`
fast-path (`:457`) sees "all valid" and **skips the texture re-upload** → the GPU samples the
PREVIOUS frame's bytes for those pages, which (correctly untiled) look like vertical-striped
garbage. It self-corrects the next frame (the master `system_page_flags_valid_and_gpu_written_` IS
cleared at `:637`, so the next swap re-marks the page invalid) → hence "varies per render".
- **Why in-match text only:** menu/front-end text is a **static** glyph atlas uploaded once → its
  pages are never invalidated → never hit the race. The in-match text path **re-rasterizes a dynamic
  glyph texture every frame** → its pages are invalidated every frame → hit the race constantly.

**The fix (`src/graphics/shared_memory.cpp`, in `patches/rexglue-sdk-current-full.patch`):** acquire
the global critical region (a recursive `std::mutex`, the SAME one `MemoryInvalidationCallback` uses)
at the top of `SetSystemPageBlocksValidWithGpuDataWritten()`, making the buffer swap atomic w.r.t.
the invalidation. One lock acquire per frame — negligible (the invalidation already takes it many
times/frame). No deadlock (recursive, single global lock, no nested acquire in the swap body).

**Verified by running (clean A/B, full version):**
- **BEFORE** (boot 10, pre-fix): the in-match HUD enemy label rendered `"GINGER ▦▦"` — "KIDS"
  replaced by vertical-striped boxes (`C:\Temp\z_pre_b10m1.png`).
- **AFTER** (boot 12, with the fix): the same label renders `"GINGER KIDS"` cleanly, and stays clean
  across ~19 captured in-match frames incl. active combat through wave 3/6
  (`C:\Temp\z_c13_hud.png`, `z_hud3.png`). Same text, same in-match dynamic renderer, pre→post.

**RE technique (the diagnostic that cracked it):** a thorough read of the runtime's texture/shared-
memory path with the key reasoning — *selective, per-quad, frame-to-frame-VARYING* corruption is the
signature of **stale/partially-uploaded data** (a skipped upload), NOT a decode/tile bug (which is
deterministic and uniform across all textures). That ruled out the entire tiling/pitch/format path
(shared with the clean menu atlas) and pointed straight at the dynamic re-upload / page-validity
bookkeeping. Confirmable A/B without code: cvar `clear_memory_page_state=false` disables the swap.

---

### (Original symptom notes below — kept for history)

**Status: OPEN (cosmetic; does not block gameplay or input).** Reported from a live
in-game capture of a tower/character info string.

## Symptom
In-game text mostly renders correctly, but a subset of glyphs is replaced by a
vertical-striped "box" artifact. Example string (Cartman's tower description):

> "CARTMAN**▟**S S**▟**E**▟**IAL ABILITY IS TO **▟**A**▟▟**ET BOMB THE A**▟**EA WITH E**▟▟**LOSI**▟**E SHELLS**▟**"

i.e. the intended text is *"CARTMAN'S SPECIAL ABILITY IS TO CARPET BOMB THE AREA WITH
EXPLOSIVE SHELLS!"* — the corrupted positions are the apostrophe, several `R/P/C/X/V`,
and the trailing `!`.

## UPDATE 2026-05-24 — isolated to IN-MATCH text; UI/menu text is fine; NOT window-size
More live captures narrowed it:
- **Menu / front-end / level-select / results text renders perfectly** (e.g. the CAMPAIGN
  LEVEL SELECT screen — "ELEMENTARY SCHOOL", "DIFFICULTY: CASUAL", "Stop the hordes attacking
  the school!" — all crisp; SETTINGS labels crisp). So the glyph atlas + the front-end text
  renderer are correct.
- **Only the in-MATCH HUD/tutorial/tooltip text corrupts** (the "use L to move / press A",
  the tower-ability banners, the in-match "PERFECT WAVE BONUS" line). That path is a
  **separate in-game text renderer** from the front-end.
- **It varies per render** — the same text can be partly readable one moment and fully
  striped the next — consistent with a **dynamic glyph-cache upload** (CPU rasterizes glyphs
  into a GPU texture per-string) whose cells are tiled/decoded wrong or read mid-upload.
- **NOT related to window size** — menu text is crisp at both the full (1296×759) and
  half (648×380) window, so it is not the present-scale/downscale.
⇒ Focus the fix on the **in-match text path's texture** (its tile mode / dynamic upload),
not the static front-end font atlas.

## Key observation (narrows the cause)
The corruption is **not consistent per letter**: e.g. `C` renders fine in "CARTMAN" but
is garbled in "SPECIAL"/"CARPET". So it is **not** a fixed bad cell in a static font
atlas (that would corrupt every instance of a given glyph identically). It is
**per-glyph-quad / positional**, which points to one of:
- a **dynamic glyph cache** (glyphs rasterized on demand into a cache texture) where some
  cells are uploaded/tiled wrong or read before upload completes (a CPU→GPU upload race or
  a tile-mode mismatch on the cache target), or
- a **render-to-texture text path** whose intermediate target is tiled/untiled with the
  wrong mode for some sub-rects.

The vertical-stripe look itself is the classic signature of a **360 tiled texture read as
linear (or wrong tile mode)** — so the data is correct but the tiling/swizzle for those
cells is misinterpreted.

## Where to look next (runtime, rexglue-sdk)
- texture tiling/untiling conversion (`src/graphics/.../texture*`, tile-mode / `GetTexture`
  / `ConvertTexture` paths) — check the font/cache texture's `xenos` tile mode vs how the
  host copy is laid out.
- any **dynamic/updated** font-cache texture (a texture the title re-uploads each
  frame/string) and whether partial-rect uploads honor tiling.
- compare a static-atlas string (menus) vs this in-match info string — if menu text is
  clean and only in-match dynamic text corrupts, that confirms the dynamic-cache path.

## Fix path (the right tool)
The runtime has a **GPU trace + texture dump**: cvar `trace_gpu_prefix=<path>` (+
`trace_gpu_stream`) records a GPU trace, and `src/graphics/trace_dump.cpp` writes textures as
PNG (`stbi_write_png`). Capture a trace of an in-match frame that shows the corrupted text,
dump the textures, find the in-match font texture, and inspect its **tile mode / format /
pitch** vs how the runtime untiles it — that pinpoints the decode bug without guessing. This is
a substantial follow-up (capture → replay/dump → compare), not a config tweak.

## Repro
Full-version run (`--license_mask=1`), enter a match, open the tower/character info for
Cartman; the description line shows the artifact. Screenshot it (host-side capture) to
inspect which glyph positions corrupt.
