# Dump analysis — *South Park: Let's Go Tower Defense Play!*

All findings below come from **read-only inspection** of the dump in the
super-repo (`South Park - Let's Go Tower Defense Play!/`). No game code or assets
are reproduced here — only structural measurements. The dump itself is
git-ignored and never committed.

## Container layout

The dump follows the Xbox 360 on-storage content layout
`<TitleID>/<ContentType>/<ContentID>`:

```
58410931/                 # Title ID = 58410931 (South Park: LGTDP!)
├── 00000002/             # content type 0x00000002 = Marketplace Content
│   ├── 6C2A0E1B…58       #   57,344 B  (LIVE STFS)
│   └── 7A56F969…58       #   57,344 B  (LIVE STFS)
└── 000D0000/             # content type 0x000D0000
    └── A7603793…58       #  924,758,016 B (~882 MiB, LIVE STFS) = main package
```

### Container format

Every file begins with the ASCII magic **`LIVE`** (`4C 49 56 45`) → they are
**LIVE-signed STFS** (Secure Transacted File System) packages. The big
`000D0000` file is the **main game package** (contains `default.xex` + assets).

The two `00000002` packages were extracted and classified (Phase 1): each holds
**one 46-byte `.bin`** — `ProfChaos.bin` and `ChallengeLevels.bin`. These are
**DLC entitlement markers** (the "Professor Chaos" and "Challenge Levels"
add-ons), *not* a Title Update — **no `.xexp`** is present. So the recompiler
ingests a single, un-patched `default.xex` (R9 closed; the two markers are
ignorable for an offline v1).

## Main package internals (read-only scan)

| Probe | Result | Meaning |
|---|---|---|
| `LIVE` magic @ 0x0 | present | LIVE-signed STFS |
| `"default.xex"` string | found @ file offset **49408 (0xC100)** | The STFS file table (data region starts at 0xC000); the game EXE is named `default.xex` as expected |
| `XEX2` magic | found @ file offset **176128 (0x2B000)** | The embedded executable is an **XEX2** image |
| Import library names | **`xboxkrnl.exe`**, **`xam.xex`** (only) | Minimal import surface — see below |

### Import surface (the key feasibility signal)

The XEX import table references **only two** system libraries:

- **`xboxkrnl.exe`** — the Xbox 360 kernel: memory (`MmAllocate…`), threads
  (`ExCreateThread`), synchronization, TLS, `Nt*` file I/O, `Rtl*` helpers,
  timing, heap. This is the core every recomp runtime implements first.
- **`xam.xex`** — Xbox Application Manager: content/storage enumeration, user
  profile, blades/notifications, sign-in, marketplace. Implemented by the
  rexglue/Unleashed runtimes (and largely stubbable for an offline title).

**Not present:** `xnet.xex`/`xonline` (networking), `xbdm.xex` (debug monitor),
`xgi`/`xhv` as separate import libs. The absence of a networking import library
is the single most encouraging measurement in this analysis — it means the
offline core does not depend on the hardest-to-emulate subsystem.

> Note: XAudio2 and Direct3D on Xbox 360 are *static* XDK libraries linked into
> the title, so they appear as **guest code**, not as import libraries. They are
> handled by the runtime's audio (XMA + SDL) and GPU command-processor layers,
> not by import shims.

## Extraction plan (Phase 1)

Goal: produce `private/default.xex` (git-ignored) plus any title-update `.xexp`.

1. **Parse the STFS** main package and extract the file table, then read out
   `default.xex` (and any other root files the loader needs). Options:
   - a small purpose-written STFS reader (block→offset with hash-block stride
     `0xAA`, data region base `0xC000`, block size `0x1000`), **or**
   - an existing extractor (e.g. wxPirs / Velocity / a QuickBMS STFS script).
   The reader path is preferred so it can live in `south-park-recomp/tools/`
   and be reproducible.
2. **Classify the two `00000002` packages**: extract and check for a
   `*.xexp` / title-update XEX. If a TU exists, it feeds XenonRecomp's
   `patch_file_path` (or rexglue's equivalent) to produce the patched XEX that
   is actually recompiled.
3. **XEX recon** on the extracted `default.xex`:
   - confirm XEX2, compression (none/basic/normal LZX) and encryption (the
     recompilers carry libmspack + tiny-AES to decompress/decrypt),
   - dump base address, entry point, section/segment table, `.pdata` presence
     (function boundaries), and the **resolved import ordinal list** — the
     concrete count of distinct kernel/xam calls = the real shim workload.
4. Record the recon output here (append a "XEX recon results" section) so the
   effort estimate in [[00-feasibility]] can be tightened from
   "library-level" to "ordinal-level".

## XEX recon results

Extracted `private/default.xex` (8,499,200 B, magic `XEX2`) and parsed its
plaintext headers with `south-park-recomp/tools/xex_recon.py`. The XEX *headers*
(security info, optional headers, import libraries) are unencrypted; only the
inner PE is compressed/encrypted, so these read without the AES key.

| Field | Value |
|---|---|
| Module flags | `0x00000001` (**TITLE**) |
| Image base | **`0x82000000`** |
| Entry point | **`0x824499A0`** |
| Image size (decompressed) | `0x930000` = **9,633,792 B** |
| PE data offset | `0x3000` · security info `0x90` · 15 optional headers |
| **Compression** | **`basic`** (block list — *not* LZX "normal") |
| **Encryption** | **`normal` (AES-128-CBC, retail)** |
| Default stack | `0x40000` (256 KiB) |
| TLS / EXECUTION_INFO / GAME_RATINGS / RESOURCE_INFO / STATIC_LIBRARIES / ORIGINAL_PE_NAME / LAN_KEY / XBOX360_LOGO | present |

**Import surface (the shim backlog, ordinal-level).** Two libraries; per-library
import-record counts read straight from the import-libraries header:

| Library | Import records | Version / min |
|---|---|---|
| `xboxkrnl.exe` | **325** | `0x20247000` / `0x20074500` |
| `xam.xex` | **162** | `0x20247000` / `0x20074500` |

≈ **487 import slots** total. That is the *upper bound* on distinct kernel/XAM
functions to provide; most already exist in the rexglue runtime, and a large part
of `xam`'s 162 are stubbable for an offline title. The exact resolved ordinal
*numbers* live in the (de)compressed PE and will be enumerated by `rexglue
codegen` in Phase 2; this count tightens the estimate from "library-level" to
"~hundreds of slots, mostly pre-implemented".

> `basic` compression + AES-128 is the easy ingest case — both rexglue and
> XenonRecomp decrypt (tiny-AES) and reassemble basic blocks natively. No `.pdata`
> / section detail is recorded here yet because it requires decompressing the PE;
> deferred to the recompiler's analysis pass (Phase 2).

## Asset inventory & engine fingerprint

Full read-only extraction of the main package → `private/extracted/` (git-ignored)
via `tools/stfs_extract.py`: **1,555 files / 46 dirs, 915,071,969 B (872.7 MiB)**.
Top-level: `media/`, `ui/`, plus root achievement/arcade PNGs, `ArcadeInfo.xml`,
`default.xex`. By type (count / size), the subsystem map the runtime must cover:

| Ext | Count | Size | What it implies for the runtime |
|---|---:|---:|---|
| `.wmv` | 177 | **651.7 MiB** | **WMV/VC-1 video** (cutscenes/intros) → FFmpeg decode + present. The single biggest payload; skippable for the core loop, real work for parity. |
| `.xwb` | 18 | 111.7 MiB | XACT **wave banks** → **XMA** decode (per-character voice banks + `StreamWaveBank` 50 MiB music). |
| `.bin` | 45 | 40.9 MiB | Level / challenge / generic binary data (engine-specific). |
| `.png` | 674 | 31.6 MiB | UI & textures as **PNG** → runtime PNG decode (not native Xbox `.dds`). |
| `.xzp` | 1 | 22.5 MiB | Packed archive ("XZP") — a custom container to crack or stream. |
| `.ttf` | 5 | 4.8 MiB | **TrueType** fonts (EN + JP `*-ja-jp.ttf`) → runtime glyph rasterization. |
| `.xsb`/`.xgs` | 3 / 1 | 0.4 MiB | XACT **sound banks** + global settings (cue/transition graph). |
| `.xmc` | 526 | 0.25 MiB | Many tiny engine containers (`movenfiredata`, `decals`, `audiobanks`, …). |
| `.lua` | 39 | 0.16 MiB | **Lua scripting** — game logic runs on an embedded Lua VM (compiled into guest code, so it recompiles "for free"). |
| `.ptc` | 1 | 0.09 MiB | Custom texture/package (`ArcadeLogo.ptc`). |
| `.updb`/`.xbv`/`.xbp` | 19 each | tiny | Per-level packaged triplets (build/version/package data). |
| `.dds` | 3 | small | A few native DDS textures. |
| `.par`/`.spm`/`.xml` | 1 / 2 / 1 | small | Misc engine data; `ArcadeInfo.xml` = XBLA arcade metadata. |

**Fingerprint summary** (Doublesix custom engine): XACT audio (XMA), **WMV
video**, PNG/TTF loaded at runtime, **Lua** for logic, custom `.xzp`/`.xmc`/`.ptc`
packaging. Audio + video are the heavyweight runtime subsystems; rendering is
modest (2.5D tower defense, mostly sprites/UI). No middleware that mandates a
separate import library (XACT/D3D/XMA are static XDK libs → guest code).

## Resolved vs. open

- **Resolved:** compression (`basic`) + encryption (AES-128); base/entry/image
  size; import libraries + counts; **no Title Update**; engine middleware map.
- **Open (deferred to recompiler analysis / later phases):** exact `.pdata`
  function count & section table (needs PE decompress, Phase 2); distinct ordinal
  *list* (Phase 2 `codegen`); `.xzp`/`.xmc`/`.ptc`/`.bin` internal formats (only
  if the loader needs them cracked vs. passed through the VFS); WMV codec exact
  profile (Phase 5 video).
