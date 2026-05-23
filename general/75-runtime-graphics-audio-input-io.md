# Runtime — graphics, audio, input, filesystem, saves

These subsystems sit beside the kernel/XAM shims (`70`) and turn the title's
guest calls into host behaviour. A mature runtime (rexglue) provides all of
them; you adapt and fill title-specific gaps.

## Graphics — command processor + render layer

- On 360, Direct3D is a **static XDK library** in the title: it builds **PM4-style
  command packets** into a ring buffer the GPU consumes. After recompilation,
  that code runs as guest code and writes packets into guest memory.
- The runtime's **command processor** parses the ring buffer and translates:
  draws, state (blend/depth/raster), resource bindings, render-target setup, and
  shader selection → into **D3D12** or **Vulkan**.
- **EDRAM / render targets / resolve:** model the 10 MB EDRAM, MSAA, fast clears,
  and the **resolve** step that copies EDRAM to a main-memory texture. Tiling /
  predicated tiling must be handled. This is a frequent source of visual bugs.
- **Shaders:** bound via the translation path in `60-gpu-shader-translation.md`.
- **Present/pacing:** map the title's swap to host present; manage frame pacing
  and v-sync.

Practical: bring it up in stages — clear screen → simple draw → UI → full scene.
On Windows prefer **D3D12** (matches the DXIL path and the host); Vulkan is the
portable alternative.

## Audio — XAudio2 + XMA

- **XAudio2** is also a **static XDK lib** in the title; the guest creates voices
  and submits buffers in guest memory. The runtime intercepts at the **audio
  driver** layer: collect submitted buffers, **decode XMA**, mix, and output via
  the host (e.g. **SDL audio**).
- **XMA** is the 360's compressed audio codec; on console it decoded via hardware
  (MMIO). The runtime decodes it in software — rexglue uses an **FFmpeg (xenia
  fork)** XMA path. PCM/ADPCM buffers pass through with format conversion.
- Watch **sample rate / channel** conversions and **buffer endianness**. A/V sync
  for any video is a separate concern.

## Input — XInput → host controllers

- The title uses **XInput** (controller state polling, capabilities, rumble).
  Bridge to the host (e.g. **SDL game controller**): map face buttons, D-pad,
  sticks (deadzones), triggers, and rumble.
- Support **multiple controllers** for local multiplayer/co-op; map user index →
  host controller. Provide a keyboard fallback for development.

## Filesystem — the VFS

- The title opens files by **device path** (e.g. `game:\…`, `d:\…`) via
  `NtCreateFile`/`NtReadFile`. The runtime's **VFS** maps these device roots to
  host directories — primarily the **extracted asset tree**.
- Handle: path translation (separators, case-insensitivity), file enumeration,
  attributes, seek/read semantics, and read-only vs writable mounts.
- Mount layout typically: the game's content root as the read-only data device,
  plus a writable **save/content device** (below).

## Saves & content

- 360 saves/content are STFS packages addressed via **`xam` content APIs**
  (`70`). For a port, back these with a **host save directory** (e.g. under the
  user's app-data) and present a single storage device.
- `XamContentCreate`/enumerate/close map to creating/listing/closing files in
  that directory. Keep a stable on-disk layout so saves survive rebuilds.
- Verify the **save→quit→relaunch→continue** loop early in Phase 5; it exercises
  content APIs, the VFS write path, and serialization endianness together.

## Bring-up order (recommended)

1. Window + present a cleared frame (graphics alive).
2. VFS mounts the asset tree (file opens succeed).
3. First real draws + shaders (something recognizable on screen).
4. Input (navigate menus).
5. Audio (music/SFX).
6. Saves (persistence).

Each step has a clear pass/fail you can observe — do them in order and record
results in the title's case-study docs.
