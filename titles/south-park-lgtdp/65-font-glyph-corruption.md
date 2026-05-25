# Rendering defect — some font glyphs render as striped/garbled boxes (2026-05-24)

**Status: ✅✅ REALLY SOLVED 2026-05-25 — `force_upload` on texture reload (verified by running).** The
2026-05-24 "FULL RESOLUTION" below (#341 full-snapshot + fast-path read lock + default
`clear_memory_page_state=false`) was **STILL insufficient**: striping RETURNED on the in-match
"HIT ENEMIES…" tutorial banner under match load (maintainer re-reported via `/remote-control`). It was
NOT a tile-mode/pitch/format/decode bug — it is a **stale page-valid state in the GPU shared-memory
upload-skip optimization**, and ultimately that optimization is **unsafe for per-frame CPU-rasterized
glyph textures**. Read the 2026-05-25 section first; the 2026-05-24 attempts follow as history.

## ✅✅ REAL RESOLUTION (2026-05-25, supersedes the 2026-05-24 "FULL RESOLUTION" below)

**The 2026-05-24 fix was *still* insufficient.** With #341's full-snapshot rebuild + the fast-path read
lock + default `clear_memory_page_state=false`, menus/results render clean — but the **in-match top
tutorial banner ("HIT ENEMIES WITH SNOWBALLS…")** still striped under match load (maintainer
re-reported via `/remote-control`, with a screenshot). A sticky-`dynamic`-bit experiment (mark a page
CPU-dynamic on its first invalidation, then never skip its upload) only **partially** helped — "HIT
ENEMIES" rendered clean but the rest stayed mush — because it inherits the same unreliable signal: a
page is only protected after the host **write-watch trips for it at least once**, and under match load
the watch misses many of the glyph texture's pages every frame.

**Deeper root cause:** `SharedMemory::RequestRanges`' per-page *"already-valid → skip upload"*
optimization is **fundamentally unsafe for a texture the CPU re-rasterizes to the same address every
frame**. Correctness there depends on the write-watch invalidating **every** changed page **every**
frame; that is a race and it loses pages intermittently → wrongly-valid pages → skipped re-upload → the
GPU samples stale bytes → striped glyphs. `clear_memory_page_state=false` only removes one *source* of
false-valid (the frame-end GPU-written propagation); it does not make the per-page skip reliable.

**The fix — force a full re-upload at the texture-reload boundary** (`shared_memory.{cpp,h}` +
`pipeline/texture/cache.cpp`): the texture cache calls `RequestRanges` **only** when a texture is
already known outdated (`base_outdated` — its own watch fired), i.e. its bytes *must* be refetched. So
add a `force_upload` parameter to `RequestRanges` and pass it **true** from the two texture-cache reload
sites (`LoadTextureData` and the batched `RequestTextures`). With `force_upload`, `RequestRanges` skips
the "all-valid" fast-path and treats every in-range page as needing upload — **except** pages the GPU
itself wrote. Vertex/index and other GPU reads keep `force_upload=false` (the skip is safe and wanted
there; this also keeps boot-time upload volume unchanged).

**⚠️ KEY LESSON — the valid-flag skip also *protects GPU-written data*, not just perf.** The first cut
of the force path used a blunt `block_valid = 0` (re-upload *everything* in range). That re-uploaded
**GPU-written pages** (resolved render targets, memexport output) from stale guest CPU memory →
clobbered the GPU's own data → the boot intro rendered **black, 6 launches in a row**. The correct force
keeps GPU-written pages valid: `block_valid = system_page_flags_valid_and_gpu_written_[i]`. CPU-
rasterized glyph pages are *not* GPU-written, so they all re-upload (fixes striping) while render
targets stay protected (fixes boot). A smaller `system_page_flags_dynamic_` sticky-bit safety net is
also kept (boot-safe; covers any CPU-dynamic page reached outside the texture-cache force path).

**Verified by running (2026-05-25, d3d12, RTX 3060; focus-free `live_input.txt`, window-capture +
zoom):** boot OK; the reported banner went **124 → 43 white/non-white transitions per row** at the same
region (dense vertical-bar striping → crisp letters); in-match HUD "GINGER KIDS" / WAVE / counters
clean; results SCORE / COIN BONUS / TOTAL clean; menus and loading hints clean. (Boot still hits the
pre-existing non-deterministic GPU-fence black-stall on *some* launches — unrelated; retry past it. The
force path is boot-safe — booted on attempt 1 once corrected.) Patch regenerated
(`patches/rexglue-sdk-current-full.patch`).

**Debugging notes:** (1) an intermittent corruption that a sticky per-page protection only *partially*
fixes is a strong tell that the protection's *trigger* (here the write-watch) is itself unreliable —
force the work unconditionally at a boundary where you *know* the data is stale (here: texture reload).
(2) Capturing a D3D window via `CopyFromScreen` requires the window foregrounded *and* content rendered;
a too-early "bright" frame is the desktop, not the game, and will de-sync scripted input.

## ✅ FULL RESOLUTION (2026-05-24, supersedes the first SOLUTION below) — INSUFFICIENT, see 2026-05-25 above

**The first fix was incomplete.** Locking the buffer swap (and later the fast-path read) reduced but
did **not** eliminate the corruption. A maintainer with the build on two PCs showed it was
**intermittent and per-visit**: different text striped each time (results TOTAL clean one visit,
SCORE/BONUS lines striped the next; menu difficulty NORMAL/HARD; the in-match hint banner) and **clean
on one machine, striped on another** — the signature of a **timing race**, not a deterministic decode
bug. **Decisive experiment:** running with `clear_memory_page_state=false` (which forces all texture
re-uploads, disabling the skip) rendered **every** screen clean — proving the bug is the *skip*, i.e.
stale page-valid state, not tiling/format.

**Root cause (confirmed by upstream rexglue-sdk issue #341, "cvar clear_memory_page_state doesn't work
as expected"):** `SharedMemory::SetSystemPageBlocksValidWithGpuDataWritten()` rebuilt the `staging`
valid-flag buffer with an **incremental per-dirty-block copy**, but `staging` is the buffer retired two
swaps ago, so its **non-dirty blocks still held stale valid bits**; copying only the dirty blocks left
those stale "valid" flags in the new active buffer → pages that should be invalid read as valid →
`RequestRanges` skips their re-upload → the GPU samples stale bytes. On NG2 (the #341 report) this lost
GPU-written validity → missing character geometry; on South Park LGTDP it left per-frame
CPU-rasterized **dynamic glyph** pages wrongly valid → striped text.

**The fix (3 parts, all in `patches/rexglue-sdk-current-full.patch`):**
1. **`SetSystemPageBlocksValidWithGpuDataWritten` — apply the #341 root-cause fix:** always rebuild
   `staging` as a **FULL** snapshot of the master flags (full `memcpy`) instead of the incremental
   copy. (The upstream issue forces `dirty_mask = UINT32_MAX`; we `memcpy` the whole buffer.)
2. **`RequestRanges` fast-path read under the global lock** (serializes every access to the valid-flag
   buffer contents — removes the lock-free read data race).
3. **`clear_memory_page_state` default OFF** (`command_processor.cpp`). Even with (1)+(2), the
   optimization still intermittently keeps *this title's* per-frame same-address glyph pages marked
   valid → stale skip → striping. Forcing re-upload (OFF) is the **only config that renders everything
   clean**, and it is the shipped default for this title. (1)+(2) keep the ON path correct for other
   titles. The cvar's "may break memory coherency" warning **did not manifest** here (a full match +
   results render correctly).

**Verified by running (default settings = optimization OFF):** the difficulty NORMAL/HARD labels, the
in-match control-hint banner, and the results SCORE/BONUS/TOTAL all render clean.

**Lesson (promoted to general/95):** for intermittent, per-visit, machine-dependent texture/text
corruption, suspect a **stale upload-skip / page-valid race**, and classify it cheaply by **forcing
all uploads** (here `clear_memory_page_state=false`) — if that's clean, it's a skip/staleness bug, not
a decoder. A *partial* lock fix that "reduces but doesn't kill" intermittent corruption is a sign the
underlying *data* (not just access ordering) is wrong — here the staging buffer's incremental rebuild.

---

### (First SOLUTION below was PARTIAL — kept for history)

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
