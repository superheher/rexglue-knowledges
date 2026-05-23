# The XEX2 executable format (what the recompiler ingests)

A 360 title's executable is a **`.xex`** (conventionally `default.xex`). XEX is a
wrapper around a standard **PE image**, adding load/security metadata, optional
headers, compression and encryption. The recompiler reads the XEX, decrypts and
decompresses the inner PE, and translates its code. You rarely parse XEX by hand
— the tools (XenonRecomp's `XenonUtils`, rexglue, `xextool`) do it — but you
must understand the fields that drive recompilation config.

## Top-level structure

```
+0x00  magic "XEX2" (0x58455832)
+0x04  module flags
+0x08  offset to PE data (the wrapped image)
+0x0C  reserved
+0x10  offset to security info
+0x14  optional header count
+0x18  optional header directory: [ (id:u32, value_or_offset:u32) × count ]
...    security info (incl. image key, region, allowed media, image size)
...    optional headers' payloads
...    PE data (possibly compressed/encrypted)
```

All multi-byte fields are **big-endian**.

### Optional headers (the directory)

Each entry is keyed by an ID; the low byte encodes how to read the value (inline
vs. offset to a payload). The ones that matter for recomp:

| Optional header | Why you care |
|---|---|
| **Base address** (image base) | Where the image maps; anchors every absolute address in the config. Commonly around `0x82000000`. |
| **Entry point** | Where execution begins. |
| **Import libraries** | List of imported libraries (name + version) and the **import records** (addresses patched at load). The recomp maps these to host shims. |
| **File format info** | **Encryption** (none / normal AES-128-CBC) and **compression** (none / basic / normal LZX) of the PE image. |
| **TLS info** | Thread-local storage layout. |
| **Original PE name / image name** | Often `default.exe`; useful sanity check. |
| **Resource info** | Embedded resources (XEX may carry `XDBF`/`SPA` etc.). |
| **System / execution info** | **Title ID**, media ID, version, base/min kernel versions. |
| **Static libraries** | XDK libs & versions linked in (engine/middleware fingerprinting). |

## Encryption & compression of the inner PE

- **Encryption.** "Normal" = AES-128-CBC. The session key is recovered by
  decrypting the XEX's stored image key with a fixed XEX key (retail vs devkit);
  the tools embed this. Result: the inner PE in the clear.
- **Compression.**
  - *None* — image stored raw.
  - *Basic* — a table of (raw size, padded/zeroed size) runs; reassemble.
  - *Normal* — **LZX** compressed in blocks (each block carries the next block's
    size + a hash). XenonRecomp bundles **libmspack** for LZX and **tiny-AES** for
    decryption; rexglue does the equivalent.

You do **not** implement these — but you record the flags during recon, because
they confirm the toolchain can ingest the file and occasionally explain odd
behaviour.

## Imports → the shim surface

The import metadata lists, per library, the **ordinals** the title uses and the
**import record addresses** in the image that the loader patches with resolved
function/variable addresses. For recompilation:

- The set of **library names** is your first feasibility read (e.g. only
  `xboxkrnl.exe` + `xam.xex` = friendly).
- The set of **distinct ordinals** is the concrete backlog of host functions to
  provide (most already exist in a mature runtime; you fill the gaps). Map
  ordinals → names via Xenia's import tables / Free60.
- Some imports are **data**, not functions (e.g. `KeTimeStampBundle`); handle
  accordingly.

## `.pdata` — function boundaries for free

The PE carries a **`.pdata`** exception/unwind directory: a table of function
**start addresses and sizes**. The analyzer uses it to recover function
boundaries directly; functions *not* in `.pdata` (e.g. leaf functions without
unwind data, or jump-table-laden functions) are found heuristically or declared
manually. See `50-cpu-recompilation.md`.

## XEXP — title updates (patches)

A **Title Update** ships an **`.xexp`** patch: a **delta** against the base XEX.
The recompiler applies it (XenonRecomp: `patch_file_path` → `patched_file_path`)
to produce the patched XEX that is actually analyzed and recompiled. If a title
has a TU, **recompile the patched image**, not the base — code addresses,
jump tables and boundaries can differ.

## What to capture during XEX recon (checklist)

Record into the title's `titles/<id>/10-dump-analysis.md`:

- [ ] magic confirmed `XEX2`; module/system flags
- [ ] **base address**, **entry point**, **image size**
- [ ] **encryption** + **compression** types
- [ ] **Title ID**, version(s), base/min kernel version
- [ ] **import libraries** + versions; **distinct ordinal list/count**
- [ ] section/segment table (code vs data ranges)
- [ ] `.pdata` present? function count
- [ ] static libraries / middleware fingerprints
- [ ] presence of a Title Update (`.xexp`)

These feed the recompiler config (`25`/`30`/`50`) and tighten the effort estimate
from "library-level" to "ordinal-level".
