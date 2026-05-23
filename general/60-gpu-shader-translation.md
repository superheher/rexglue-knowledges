# GPU & shader translation — Xenos → DXIL/SPIR-V

The runtime does **not** emulate the GPU; it translates the **command stream**
into D3D12/Vulkan and translates **shaders** from Xenos microcode to HLSL, then
to DXIL (D3D12) or SPIR-V (Vulkan) via DXC. Much of this is title-specific —
expect to fix shaders case-by-case. XenosRecomp's documentation is the reference
for the gotchas below; rexglue performs equivalent translation in its renderer.

## Two ways to provide shaders

1. **Prebuilt cache** (XenosRecomp style): scan the XEX/assets for shader blobs,
   translate + compile them, and emit a **cache** keyed by **XXH3 hash**; at
   runtime the game's shader bytes are hashed to look up the recompiled version.
   SPIR-V is `smol-v`-encoded then zstd-compressed; DXIL stored as-is. Good for
   shaders embedded in the executable or uncompressed archives.
2. **Runtime translation** (rexglue style): the renderer translates shaders as it
   sees them via the command processor. Fewer build-time steps; correctness is
   handled in the live renderer.

## Shader container & reflection

Xenos shaders ship in a container with **constant-buffer reflection,
definitions, interpolators, vertex declarations, and instructions.** The
recompiler relies on the **reflection** to populate constant registers — **if
reflection is missing, translation fails** (workaround: declare a `float4` array
covering the whole register range). The container is reverse-engineered "just
enough" for known titles; new titles may need more.

## Instruction translation

- **Vector/ALU** ops translate directly and usually work.
- `INF`/`NaN` handling may be imperfect (often clamped to `FLT_MAX`); verify in
  edge cases.
- **Dynamic register indexing** is commonly **unimplemented** — the fix is to
  model registers as an array the shader indexes (instead of separate locals).
- **Dynamic constant indexing** across multiple operands can misbehave.

## Constants

- **Vertex** constants ≈ **256× `float4`** (4096 B); **pixel** ≈ **224× `float4`**
  (3584 B); plus a runtime-specific **shared** constants buffer.
- Copied from the guest render device; shaders expect **little-endian** constants.
- **Boolean** constants are packed into a 32-bit int in the shared buffer (bit N
  = bool N); 360 supports up to 128 — widen the type if a title needs more.
- **Integer constants are unimplemented** upstream — add slots if required.
- Implemented as root constant buffers (D3D12) / push-constant GPU addresses with
  `vk::RawBufferLoad` (Vulkan); **out-of-bounds dynamic reads must return 0** —
  enforced by clamping the index and zeroing OOB.

## Vertex fetch & formats (a top source of title-specific work)

- Vertex declarations are converted to **native input layouts** (rather than
  generating fetch shaders per declaration). This avoids runtime shader
  permutations but is less flexible.
- **Endian swap** of vertex data (treating buffers as `u32` arrays) **swizzles**
  sub-32-bit elements: 8-bit usually fine; **16-bit gets swizzled to YXWZ** and
  must be corrected (a per-`TEXCOORD` "needs re-swizzle" bitmask in shared
  constants). Other semantics may need the same treatment per title.
- **`R11G11B10`** vertex format is **unsupported on desktop**; unpacked manually
  in the vertex shader (often only for `NORMAL`/`TANGENT`/`BINORMAL`) via a
  specialization constant.
- Some semantics may need to be **`uint4` instead of `float4`** for specific
  shaders (title-specific).
- **Instancing** is done manually on 360 (e.g. index buffer fed as a vertex
  stream, instance index from a constant) — **completely game-specific**; you
  must map the title's scheme onto modern instancing.
- Vulkan vertex **locations** may be hardcoded for a specific title and limited
  to 16; a generic port assigns unique locations per vertex shader.

## Textures & samplers

- **Bindless**: descriptor indices live in the shared constant buffer, with
  separate indices per texture type to avoid mismatches.
- Many sampler features (explicit LOD, filter selection) are **unimplemented**
  upstream — only what known titles needed (e.g. pixel offset for shadows).
- **Cube maps**: the `cube` instruction's face/coord computation is emulated by
  storing directions in a local array indexed at fetch time, which DXC optimizes
  away for simple shaders; complex shaders may defeat this and need the exact
  hardware computation.
- Some 360 sampler types have no desktop equivalent and need special handling.
- **1D textures** often unimplemented (easy to add).

## Control flow

- HLSL has no `goto`, so shader control flow becomes a **`while(true) switch(pc)`**
  state machine; simple shaders are **flattened** so DXC can optimize.
- Complex control flow is under-tested upstream — fixable, but find a repro first.

## Specialization constants

- Used for behaviours toggled at pipeline-creation: e.g. the `R11G11B10`-unpack
  flag, and **alpha test** (desktop has no fixed-function alpha test, so a
  "discard if `< threshold`" is appended to the pixel shader; add comparison
  modes per title).
- **Vulkan** has native spec constants; **DXIL does not** — emulated by compiling
  shaders as **libraries** with an unimplemented "get spec constant" function
  that the runtime implements and **links** in to produce the specialized binary
  (the "DXIL linking" technique).

## Known-unimplemented (watch for these)

- **Memory export** from shaders.
- **Point size.**
- Mini vertex-fetch instructions / fetch bindings.
- ...and likely more for any given title — treat the translator as a starting
  point you extend.

## Bring-up tactics

- Get **something on screen** first (even wrong colors); then fix per shader.
- Compare against **reference footage** for each screen/effect.
- When a shader is wrong: dump its bytes/hash, find the translated HLSL, and
  inspect the specific instruction/format/semantic involved.
- Record each fix (hash → symptom → cause → fix) in the title's
  `40-render-notes.md`.
