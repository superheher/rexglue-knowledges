# Xbox 360 hardware — what a recompiler must reproduce

Only the parts that affect recompilation are covered. Authoritative detail lives
in Xenia and Free60 (see `99-references.md`).

## CPU — "Xenon"

- **3 physical cores, 2 hardware threads each = 6 logical threads**, ~3.2 GHz,
  **in-order**, shared 1 MB L2.
- **PowerPC** 64-bit ISA (custom, close to PPC970/Cell PPE family), **big-endian**.
- **VMX128** vector unit: an extension of AltiVec/VMX with **128 vector
  registers** (vs 32), used heavily by games for SIMD math and by the XDK's
  graphics helpers.

### Recompilation consequences
- **Endianness.** Guest memory is big-endian; the host is little-endian. The
  recompiler **byte-swaps on every guest memory load/store** and marks them
  volatile to stop unsafe reordering. Vector registers are handled by **reversing
  the whole 16-byte vector**, so vector instructions must account for reversed
  element order (e.g. dot-product component order, pack/unpack argument order).
- **32-bit guest pointers.** The title addresses memory as 32-bit. The runtime
  allocates one big **base pointer** and guest addresses are offsets into it;
  generated functions receive `(ctx, base)`.
- **VMX128 → host SIMD.** Vector ops are emitted as x86 intrinsics (SSE/AVX) or,
  on ARM64, via SIMDe. Some packed/unpacked D3D formats are only partially
  implemented and warn when hit.
- **Floating point semantics.** The scalar FPU **keeps denormals**, while VMX
  **flushes denormals to zero**. The recompiler tracks FP state and toggles
  denormal flushing per-instruction as needed. Getting this wrong = subtle math
  bugs.
- **CPU state struct ("PPC context").** Every register lives in a struct passed
  to every function. Many registers can later be optimized into locals once the
  port is stable (see `50-cpu-recompilation.md`).
- **In-order timing.** Functionally irrelevant after translation, but games may
  have timing assumptions; pacing is a runtime concern, not a codegen one.
- **Threads.** The 6-thread model maps onto host threads; the runtime must
  provide thread creation, affinity (often ignorable), TLS, and sync primitives
  matching the kernel's semantics.

## GPU — "Xenos"

- ATI-designed **unified shader** GPU (ancestor of the TeraScale architecture).
- **10 MB EDRAM** holds render targets / depth and does MSAA + fast clears;
  results are **"resolved"** from EDRAM back into main-memory textures.
- **512 MB unified GDDR3**, shared between CPU and GPU.
- Drawing is driven by a **command buffer** (PM4-style packets) the CPU writes
  and the GPU consumes — the runtime's *command processor* parses these.
- **Shaders are microcode** in a reflection-bearing container (constants,
  interpolators, vertex declarations, instructions). Translated to HLSL→DXIL or
  SPIR-V (see `60-gpu-shader-translation.md`).

### Recompilation consequences
- You do **not** emulate the GPU gate-by-gate. The runtime **translates the
  command stream** into modern D3D12/Vulkan calls and **translates shaders** to
  DXIL/SPIR-V. This is "render-layer HLE."
- **EDRAM / tiling** must be modelled: render targets, MSAA resolve, predicated
  tiling, and EDRAM aliasing. This is a common source of visual bugs.
- **Vertex fetch** on Xenos is programmable; declarations are converted to native
  input layouts (or fetched from buffers). 8/16-bit and packed formats
  (e.g. `R11G11B10`) need explicit handling.
- **Constants** are register files (vertex ~256× `float4`, pixel ~224× `float4`),
  uploaded as constant buffers / push constants.

## Memory & address space

- **512 MB unified RAM.** Title virtual addresses are commonly based around
  `0x82000000` for the executable image; the heap and other regions follow.
- The recomp uses a single host allocation as the **guest physical/virtual base**;
  indirect-call resolution and "function pointer" tables rely on guest addresses
  mapping deterministically into this base (XenonRecomp places function-address
  lookup tables just past the valid XEX region — see `50-cpu-recompilation.md`).

## Kernel & system software (HLE targets)

The title links the **XDK**; at runtime it imports from:
- **`xboxkrnl.exe`** — kernel: memory (`MmAllocatePhysicalMemory`, …), threads
  (`ExCreateThread`), sync (events/mutants/semaphores), TLS, `Nt*` file I/O,
  `Rtl*` helpers, timers/clocks, heap.
- **`xam.xex`** — Application Manager: UI/blades & notifications, content &
  storage enumeration, user profile / sign-in, marketplace.

XAudio2 and Direct3D on 360 are **static XDK libraries** compiled *into* the
title — they appear as **guest code**, not imports, and are serviced by the
runtime's audio and GPU layers rather than by import shims. See
`70-runtime-kernel-and-xam.md` and `75-runtime-graphics-audio-input-io.md`.
