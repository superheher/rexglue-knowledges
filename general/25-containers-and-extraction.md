# Containers & extraction — getting `default.xex` and the assets

A dump is rarely a bare XEX; it is a **container** holding the executable plus
the title's asset files. You must extract both: the **`default.xex`** (to
recompile) and the **asset tree** (for the runtime's virtual filesystem to mount
at run time). Identify the container by its **magic / structure**.

## Container types you'll meet

### STFS (XBLA, DLC, saves, updates)
*Secure Transacted File System* — the package format for Xbox Live Arcade games,
DLC, title updates, profiles and saves. Signing variants by the **magic at
offset 0**:

| Magic | Meaning |
|---|---|
| `CON ` | Console-signed (by a specific console) |
| `LIVE` | Xbox Live-signed (official downloads) |
| `PIRS` | Microsoft offline-signed (system updates, some content) |

Structure essentials (for read-only extraction you can ignore the signatures):
- **Header region** spans the first `0xC000` bytes (license, metadata, the **volume
  descriptor**, package name, title/content metadata, thumbnail).
- **Data region** begins at **`0xC000`**; **block size `0x1000`** (4 KiB).
- A **file table** (entries of `0x40` bytes: name, flags, block count, starting
  block, file size) describes the contents; the volume descriptor points to it.
- **Hash blocks** are interleaved: one level-0 hash block per **`0xAA` (170)**
  data blocks, with higher-level hash blocks above them; the read-only vs
  read-write flag changes backup-hash spacing. **Block→offset conversion must
  skip these hash blocks** — the one piece of real arithmetic in an extractor.

#### Block → offset (the one real computation, validated)

For a logical data block `b`, the byte offset in the package is:

```
f = 1 if (blockSeparation & 1) else 2     # hash-table copies: 1 read-only, 2 read-write
backing(b) = b + f * ( b//0xAA + b//0x70E4 + b//0x4AF768 )
offset(b)  = 0xC000 + backing(b) * 0x1000
```

`0xAA`=170 (L0 stride), `0x70E4`=170² (L1), `0x4AF768`=170³ (L2). Equivalent
"hash-before" formulations exist (Velocity's `ComputeBackingDataBlockNumber` with
base `0xB000`); they differ by a constant `+1`/`−0x1000` that cancels — pick one
and **validate against anchors**: e.g. file-table block `0` → `0xC000`, and the
known `default.xex` start block → its `XEX2` magic. Volume-descriptor fields
needed: `blockSeparation` (bit0), `fileTableBlockNumber`, `fileTableBlockCount`.

Each file-table entry's flag byte has **bit6 = "contiguous"**: when set (the
common case for official read-only packages) you can read sequential *logical*
blocks straight through `offset(b)`; when clear, follow the per-block next-block
links stored in the L0 hash entries (`0x18`-byte entries: status `@0x14`,
next-block u24 BE `@0x15`). A reusable Python implementation:
[[../titles/south-park-lgtdp/10-dump-analysis|the South Park port]]
ships `tools/stfs_extract.py`.

> An XBLA game is typically a single **`LIVE`** package whose root contains
> `default.xex` plus the title's files. Title updates and DLC arrive as separate
> (often small) STFS packages — classify each.

### GOD — "Games on Demand" / installed XBLA
STFS-based but **split**: a small header file plus one or more **`Data####`**
chunk files in a sibling directory. Conceptually the same filesystem spread over
multiple files; extractors that understand GOD reassemble it.

### ISO / GDFX (disc games)
Disc titles are **XGD** images using the **XDVDFS/GDFX** filesystem. The root
contains `default.xex` and assets. Extract with **`extract-xiso`** (also useful
for rebuilding/trimming). Disc images may have a video partition + a game
partition at a known offset.

## Tools

| Tool | Handles | Notes |
|---|---|---|
| **wxPirs** | STFS | Simple GUI extractor for CON/LIVE/PIRS packages |
| **Velocity** | STFS / packages | Cross-platform package manager/extractor |
| **Horizon** | STFS | Windows package editor/extractor |
| **QuickBMS** + STFS script | STFS | Scriptable/CLI extraction |
| **extract-xiso** | ISO/GDFX | Disc image extraction & rebuild |
| **xextool / xex utilities** | XEX | Inspect/decompress XEX after extraction |
| **a purpose-written reader** | STFS | Preferred for reproducibility in `tools/` |

For an automated/agent pipeline, a **small committed extractor** (Python or C++)
beats a GUI tool: it is reproducible and scriptable. STFS read-only extraction is
modest (parse volume descriptor → walk file table → for each file, follow its
block chain, converting block numbers to offsets while skipping hash blocks).

## Extraction goals (per title)

1. **`default.xex`** → a git-ignored `private/`/`extracted/` dir. Verify the
   first four bytes are `XEX2`.
2. **Asset tree** → a content root the runtime mounts via its VFS. Preserve the
   original directory layout and names (the guest opens files by path).
3. **Title updates / DLC** → extract and classify; keep any **`.xexp`** for the
   recompiler's patch input (see `20-xex-format.md`).
4. Note total sizes and notable file types (audio banks, video, archives) — they
   hint at which runtime subsystems you'll need (XMA, video, custom archive
   formats).

## Legal & hygiene

- **Bring your own dump.** Extract only from content you are entitled to use.
- **Never commit** extracted code or assets — git-ignore the dump, `private/`,
  `extracted/`, `*.xex`, `*.xexp`, and the asset root.
- Record *measurements* (sizes, offsets, formats) in the KB, not *content*.
