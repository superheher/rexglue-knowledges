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
`000D0000` file is the **main game package** (contains `default.xex` + assets);
the two `00000002` files are small **Marketplace Content** packages (candidate
title update / DLC / avatar award — to be classified during extraction; see R9
in [[00-feasibility-analysis]]).

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
   effort estimate in [[00-feasibility-analysis]] can be tightened from
   "library-level" to "ordinal-level".

## Open questions to resolve during extraction

- Exact XEX compression/encryption flags (affects nothing functionally — the
  tools handle it — but good to record).
- Whether a title update exists and changes code (jump tables/boundaries).
- Distinct imported ordinal count (drives the kernel/XAM shim backlog).
- Engine markers inside the (decompressed) image: any third-party middleware,
  scripting VM, or audio/video middleware that needs its own handling.
