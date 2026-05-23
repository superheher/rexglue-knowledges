# Runtime — kernel (`xboxkrnl`) & app manager (`xam`)

The translated CPU code calls into the 360's system libraries; the runtime must
provide host implementations ("shims"). Mature runtimes (rexglue; the Unleashed
runtime) already implement most of these — your job is to **fill the gaps the
title hits** and fix mismatches. This is "high-level emulation" of the OS.

## How imports are wired

- Each guest import is a **(library, ordinal)** pair. Map ordinal → name (Xenia's
  import tables / Free60), then bind it to a host function.
- A shim receives arguments per the PPC ABI (via the context/registers) and reads
  /writes **guest memory through the base pointer**, honouring **big-endian**
  field order on every struct it touches.
- Maintain a **backlog**: list every import the title references; mark
  done/stub/todo (`titles/<id>/20-imports-backlog.md`).

## The incremental-stub method (how to move fast)

1. Run; hit an unimplemented import → it **logs name + args** and ideally
   continues with a plausible default.
2. Decide per call: **implement**, **stub-succeed** (return a benign success and
   sane out-params), or **stub-fail gracefully** (for optional/online features).
3. Prefer "succeed plausibly" so execution proceeds to the next real problem.
4. Implement properly once a stub causes a visible issue.

> The fastest path to first-frame is a wall of well-chosen stubs, not a complete
> kernel. Correctness comes in the bring-up loop.

## `xboxkrnl.exe` — subsystems

| Area | Representative APIs | Host mapping |
|---|---|---|
| **Memory** | `MmAllocatePhysicalMemory(Ex)`, `NtAllocateVirtualMemory`, `MmGetPhysicalAddress`, `MmFreePhysicalMemory` | Carve from the guest base allocation; track regions; honour alignment/large pages |
| **Threads** | `ExCreateThread`, `KeSetAffinityThread` (often ignorable), thread suspend/resume, `NtResumeThread` | Host threads; map the 6-thread model; affinity usually a no-op |
| **Sync** | events/mutants/semaphores, `KeWaitForSingleObject`/`Multiple`, `KeSetEvent`, critical sections | Host primitives behind the kernel **dispatcher object** model; getting wait semantics right matters |
| **TLS** | `KeTlsAlloc/Get/Set/Free` | Host TLS |
| **Time** | `KeQuerySystemTime`, `KeQueryPerformanceCounter`/`Frequency`, interrupt time, `KeDelayExecutionThread` | Host clocks; keep frequency consistent with what the title expects |
| **File I/O** | `NtCreateFile`, `NtReadFile`/`Write`, `NtQueryInformationFile`, `NtClose` | Route to the **VFS** (`75`) |
| **Rtl helpers** | `RtlInitializeCriticalSection`, `Rtl*Memory`, unicode/string, `RtlNtStatusToDosError` | Direct host implementations |
| **Heap** | `RtlAllocateHeap`/`Free`/`Create` | Host heap over guest memory |
| **Debug** | `DbgPrint`, `RtlRaiseException`(careful), `HalReturnToFirmware` | Log; map shutdown/reboot to app exit |
| **Misc** | `XexGetModuleHandle`, `XexGetProcedureAddress`, `KeBugCheck` | Module/proc resolution within the recomp; bugcheck → fatal log |

Notes:
- **Endianness:** every multi-byte field written into a guest struct (e.g.
  `LARGE_INTEGER`, handles, sizes) must be in **big-endian** as the guest reads
  it. A huge fraction of "kernel returns garbage" bugs are missing swaps.
- **MMIO:** some hardware (e.g. XMA decode) was driven via MMIO on console;
  XenonRecomp does **not** implement MMIO — the runtime provides the function
  (e.g. XMA decode) at a higher level instead (`75`).

## `xam.xex` — application manager

Largely **stubbable for an offline title**, but the title will call it during
init and menus.

| Area | Representative APIs | Host mapping |
|---|---|---|
| **Content/storage** | `XamContentCreate(Ex)`, `XamContentClose`, enumeration, `XamContentGetDeviceData` | Map to a host **save/content directory**; present one storage device |
| **User/profile** | `XamUserGetXUID`, `XamUserGetSigninState`, gamertag, `XamUserGetSigninInfo` | A single **offline signed-in user** |
| **Notifications/blades** | `XNotifyGetNext`, `XNotifyDelayUI`, listeners | Drive a minimal notification queue; mostly no-ops |
| **System UI** | `XShowKeyboardUI`, message boxes, sign-in/marketplace UI | Host dialog or auto-dismiss/stub |
| **Input glue** | XInput-style enumeration (sometimes via xam) | Bridge to host controllers (`75`) |
| **Achievements/marketplace/LSP** | enumerations, `XamShow*` | Stub to "unavailable"/empty |

Strategy for `xam`: **succeed with "offline, one user, one storage device, no
network."** That satisfies most init paths without implementing real services.

## Validating shims

- Log the **first** call to each import (name, args, returned value) — this
  builds your backlog automatically.
- For structs, dump fields and verify endianness against the title's behaviour.
- Keep the backlog doc current; it is directly reusable for the next title that
  shares imports.
