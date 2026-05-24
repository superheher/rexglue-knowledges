# 70 — Movie playback (intro + cutscene .wmv): the premise was outdated — the GUEST already plays them

**Headline (2026-05-24, verified by running):** the intro + level cutscene `.wmv` movies are
**NOT black & silent** in the current build. The title's **own in-software WMV3 (VC-1) video +
WMA2 audio decoders already render them** — video *and* audio. The earlier "black & silent (no
WMV3/WMA2 decoder)" note (doc 67 §4) was an **assumption that was never screenshot-verified**; the
video was likely black *earlier* only because of the GPU shared-memory page-validity bug (doc 65),
which has since been fixed — that fix also repaired the movie's video-texture upload.

A full cross-platform libavcodec decode path was built + proven during this investigation, then
**reverted** (maintainer scope decision) because the guest already decodes the movies. See §4 for
how to bring it back if the guest path ever regresses.

---

## 1. The premise, and how it was disproven (verify-by-running)

The goal was "make the black & silent movies render video+audio with libavcodec". Before building
a fix, the movie path was reverse-engineered, and the *actual* on-screen state was checked:

- **RE:** the title imports **no system video-decode API** (no XMV/XMedia/XMP-movie/
  `XamLoadExtraAVCodecs`); video imports are only `XGetVideoMode`/`XGetAVPack`/`Vd*`, audio is
  `XMACreateContext`/`XAudio*`. It opens `…\Movies\en-en\sp_xbox_0_intro.wmv` and **reads the whole
  file in ONE `NtReadFile`** (`len=0x816335` → guest buf `0x46640030`), then decodes from memory.
  The movie is a **scene node** ticked each frame (chain: `xstart → … → sub_82150970 → sub_822132A0
  scene-tree → sub_824267B0/sub_82426298 movie node, 0x82426xxx → … → sub_82447E98 open`). Found via
  `[MOVIE]` instrumentation in `NtCreateFile`/`NtReadFile` + a host backtrace (host stack == guest
  stack) symbolized with cdb `ln south_park_td+OFFSET`.
- **Decisive tests (screenshots):**
  - **Mid-movie screenshot:** real, *animating* intro video (Cartman, then a town pan) with the
    game's own "Ⓐ SKIP" prompt; then it **completes and advances to the title** — no hang.
  - **Rename the `.wmv` aside →** the movie scene goes **BLACK + skip prompt** (and the game retries
    opening the file). ⇒ the game itself decodes the file when present.
  - **My libavcodec overlay never actually ran** (it logged nothing in any run — the `Presenter` is
    null at `OnInitialize`, so registration was skipped and `Start()` returned early), yet the video
    showed in every run, **including with the overlay forced off** (`--movie_playback=0`). ⇒ the
    video is the **guest's**, not the overlay's.
  - **Audio:** `--audio_dump` captured a **~26 s music segment** (peak 0.43) exactly at the movie's
    time window; cross-correlation against the WMV's WMA2 track peaks at the probe's offset (the
    dumped audio is time-aligned with the WMV music). ⇒ the guest also decodes the **audio**.
- **Cutscenes:** `Level*Intro/Mid/End.wmv` are the **identical format** (WMV3 Main + WMA2, 1280×720,
  24 fps) played by the **same scene-node movie system** ⇒ they decode the same way.

**Conclusion:** intro + cutscenes already play real video + audio via the guest decoders. Likely
fixed incidentally by doc 65 (the GPU page-validity fix repaired the decoded-frame texture upload;
the in-software decoders were always present).

---

## 2. THE LESSON (promoted to general/95)

**Reproduce/observe the actual broken state BEFORE building a fix — don't trust a derived premise.**
"No decoder imported → black" was a *reasonable-sounding inference* that was simply **false** (the
decoder is statically linked in-guest, not a stubbed API). One mid-movie screenshot + one
rename-the-asset test would have caught it in minutes. A large FFmpeg VC-1/WMV3 + libavformat build
and a host overlay were written before the screenshot was taken. Cost: a session of avoidable work.
Corollary: when something is "documented as broken" but the doc predates other fixes, **re-verify by
running** — an unrelated fix (here doc 65's page-validity fix) may have already resolved it.

---

## 3. ffprobe of the movies (for reference)

```
sp_xbox_0_intro.wmv : asf; Video wmv3 (Main) yuv420p 1280x720 24fps; Audio wmav2 44100Hz 2ch; ~22s
Level10End.wmv      : asf; Video wmv3 (Main) yuv420p 1280x720 24fps; Audio wmav2 44100Hz 2ch; ~26s
```
All `…\Movies\en-en\*.wmv` are WMV3+WMA2/ASF (the runtime's locale-fallback serves the en-en path).

---

## 4. The libavcodec path that was built + proven, then reverted (how to restore)

A working, cross-platform host decoder was built and **proven to decode the real intro** (a saved
frame was a correct Cartman-in-town image; `tools/movie_decode_test.c`), then **reverted** since the
guest already plays the movies. If the guest path ever regresses and a host decoder is wanted again,
the recipe is recorded here:

- **FFmpeg build (audio-only → +video):** `ff_wmav2_decoder` (WMA2) was already enabled; add the
  **VC-1/WMV3 decoder + VC-1 parser** + transitive deps (from FFmpeg `configure`'s `_select` graph:
  `mpegvideo`, `h263`/`mpeg4video`, `msmpeg4`, `intrax8`, and DSP helpers `blockdsp/h264chroma/
  h264qpel/hpeldsp/qpeldsp/me_cmp/pixblockdsp/videodsp/startcode/vc1dsp/wmv2dsp`), plus the matching
  `libavcodec/x86/*_init.c` C-fallback stubs (HAVE_X86ASM=0 → the generic `.c` still call
  `ff_*_init_x86()`). Flip the 21 `CONFIG_*` flags 0→1 in `thirdparty/FFmpeg/config_*.h`. Register
  `ff_vc1_decoder`/`ff_wmv3_decoder` in the `ffmpeg-overlay/codec_list.c`, `ff_vc1_parser` in
  `parser_list.c`. **Demux:** add a `libavformat` target (minimal core + `asfdec_f`/`asf`/`asfcrypt`/
  `avlanguage` + `riffdec`; **no file:// protocol** — `file.c`/`file_open.c` pull POSIX
  `read`/`write`/`open` that don't link under clang-msvc; feed a **custom AVIO** from the in-memory
  `.wmv` bytes). Add `oldnames` to libavutil/libavformat link (POSIX `fileno`/`isatty`). YUV420P→RGBA
  manually (no swscale).
- **Player:** `rex::ui::MoviePlayer` decode thread (custom AVIO → VC-1→RGBA paced to wall-clock +
  WMA2→its own SDL stream), presented as a `UIDrawer` fullscreen overlay via the cross-platform
  `ImmediateDrawer` (register it **after** the presenter exists — it's null at `OnInitialize`; that
  init bug is why the overlay never ran this session). Start from the `.wmv` read hook; end on
  EOF/skip (A polled via `input_system`). Gate behind `--movie_playback`.
- The full diff was captured during the session; it is not committed (reverted per scope decision).
