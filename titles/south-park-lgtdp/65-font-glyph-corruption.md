# Rendering defect — some font glyphs render as striped/garbled boxes (2026-05-24)

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

## Repro
Full-version run (`--license_mask=1`), enter a match, open the tower/character info for
Cartman; the description line shows the artifact. Screenshot it (host-side capture) to
inspect which glyph positions corrupt.
