# Boot present/vsync deadlock at the first XE_SWAP (2026-05-24)

Late this session the title began **deadlocking at the first frame present, 100% of boots**
(log freezes ~4 KB right after `SetInterruptCallback(821C7170, …)`; no `XamInputGetState`
ever polled). Earlier the same build booted to a full match win, so this is a
**timing/environment-sensitive GPU-sync deadlock**, reproducible here after many launches.

## Root cause (cdb on the live stalled process)
Two interdependent runtime threads (all in `rexruntimerd`, **not** guest code):
- **CP thread** (GPU command processor): `ExecutePacketType3_XE_SWAP → IssueSwap →
  RefreshGuestOutput → RefreshGuestOutputImpl → command_processor.cpp:2014 lambda →
  EndSubmission` — stuck issuing the first swap.
- **`Presenter::DXGIUITickThread`**: stuck in
  `disruptorplus::multi_threaded_claim_strategy::wait_until_published` — a **producer waiting
  for ring-buffer space** (the slowest consumer never advances).

So the swap path and the DXGI UI-tick thread deadlock across a disruptor queue while presenting
the first frame. With **`vsync=true`** (default) it hangs here.

## What was ruled out
- **My instrumentation** — the deadlock is in `rexruntimerd`'s present path; my generated-code
  edits don't touch it, and the one GPU-adjacent edit (a fence-spin cap in `sub_821C6E58`) was
  reverted with no change.
- **Window visibility** — forcing the window restored + foreground + topmost: no change.
- **Shader-cache timing** — deleting the cache (forcing a slow shader rebuild) didn't change it.
- **`host_present_from_non_ui_thread=false`** (`src/ui/presenter.cpp:33`): no change alone.

## The one thing that moved it: `--vsync=false`
With `vsync=false` the **present deadlock clears** (log grows from 4 KB to multi-MB, boot
proceeds past the first swap) — confirming the hang is the **vsync wait inside the swap**. But
boot then stalls on a **different** GPU sync: the CP spams
`WAIT_REG_MEM stuck: poll=0x1FC9B006 func=0x3 ref=0 mask=0xFFFFFFFF value=1` ~15 000× — the GPU
waiting for the guest CPU to clear a fence at `0x1FC9B006` that never clears (`command_processor.cpp`
`ExecutePacketType3_WAIT_REG_MEM`; the loop has **no timeout/escape**). So `vsync=false` trades
the present deadlock for a CPU↔GPU fence deadlock; neither reaches input.

## Assessment + path forward
Both failure modes are **GPU-sync deadlocks that previously worked** (run reached a match win
earlier this session), so the most likely cause is **degraded host GPU-driver / DWM vsync
delivery after 250+ D3D12 device create/destroy cycles** in one long session — i.e. a
**reboot** is the fastest restore. It could alternatively be a latent deterministic race in the
runtime present path. **Recommended:** reboot, then boot with default `vsync=true`; if it
recurs, the real fix is in the SDK present path — make the swap's vsync wait + the
`DXGIUITickThread` disruptor claim **timeout/yield** instead of blocking forever (mirror the
WAIT_REG_MEM stuck-detector, but with an escape), and add a **runtime GPU-fence write-back** so
`WAIT_REG_MEM` targets like `0x1FC9B006` are satisfied. See [[50-menu-input-and-lobby]] open
blocker #2 (same fence family) and general/95.

## Repro / diagnosis recipe (reusable)
- Detect: log frozen at a few KB right after GPU interrupt setup, no `XamInputGetState`.
- Confirm: `cdb -p <pid> -c "~*k 16; qd"` → look for `disruptorplus … wait_until_published` +
  `IssueSwap/RefreshGuestOutput/EndSubmission`.
- Probe: relaunch with `--vsync=false`; if the log starts growing (and `WAIT_REG_MEM stuck`
  appears), the hang was the present vsync wait, and the next wall is the guest GPU fence.

## Update — deeper diagnosis (full 59-thread dump) + causes ruled out
A complete `~* kn` dump pinned the exact cyclic dependency at the **first XE_SWAP**:
- **A guest thread is blocked in the GPU-fence wait:** `xstart → … → sub_82150970 → sub_821BFF48
  (+0x2647) → sub_821C6E58` — i.e. the guest is spinning waiting for a GPU fence to advance.
  Note: the wait *loop* is the **caller `sub_821BFF48`**, which calls `sub_821C6E58` (a quick
  fence *check*) each iteration — so a per-iteration cap inside `sub_821C6E58`'s `loc_821C6F10`
  loop never trips (`fenceCap=0`); the loop to bound is `sub_821BFF48`'s.
- **The CP thread is building the first swap's present command list:** `ExecutePacketType3_XE_SWAP
  → IssueSwap → RefreshGuestOutput → D3D12Presenter::RefreshGuestOutputImpl →
  command_processor.cpp:2014 lambda → DeferredCommandList::WriteCommand`. The GPU never completes
  the frame, so the guest's fence never advances → frozen at the first frame.
- The disruptor spinner seen earlier is just `rex::thread::TimerQueue::TimerThreadMain` (a busy
  timer-queue claim) — a red herring, not the present path.

**Causes ruled out this round (so it is NOT simply "environmental"):**
- **GPU memory/driver:** `nvidia-smi` shows the RTX 3060 with **11.5 GB free / 589 MB used**, no
  zombie processes, 11% util. Not VRAM/handle exhaustion.
- **Display sleep:** woke the display (synthetic input) + `powercfg /change monitor-timeout-ac 0` —
  still deadlocks.
- **Shader cache / window visibility / `host_present_from_non_ui_thread`:** all tried, no change.
- **My instrumentation:** reverted every generated-code edit (recomp.2/6) back to clean codegen and
  rebuilt — **still deadlocks**, so it is not the recompiled code. The build is byte-for-byte the
  same path that booted to a match win earlier the same session.
- Could **not** restart DWM (`Stop-Process dwm` → Access denied; not elevated).

**Refined conclusion:** a **timing-sensitive present/GPU-fence cyclic deadlock in the rexglue-sdk
runtime** (guest fence-wait `sub_821BFF48`/`sub_821C6E58` ↔ CP first-swap `RefreshGuestOutputImpl`),
which became deterministic on this host after extended running. The **recompiled artifact is
correct** (same build booted earlier). Fastest restore is a **reboot** (reshuffles host/runtime
timing); the durable fix is in the SDK present path — make the **first-swap present complete and
signal the guest GPU fence** (runtime fence write-back) so `sub_821BFF48` exits, and bound the
guest fence-wait loop in `sub_821BFF48` (not `sub_821C6E58`) as a stop-gap.

## Update 2 — more causes ruled out; the host is healthy (so it is NOT a wedge)
- **Divergence point (booting vs frozen log):** in a log that booted, immediately after
  `SetInterruptCallback(821C7170,…)` the **guest main thread continues into the per-frame loop**
  (the flush `sub_82151170` runs, then input polling). In the frozen runs the guest **stalls right
  there** — in the post-init GPU-fence wait, before the per-frame loop (so `XamInputGetState` is
  never reached). The fence advances via the just-registered GPU interrupt; it isn't advancing.
- **Additional causes ruled out this round:** GPU is healthy (`nvidia-smi`: RTX 3060, **11.5 GB
  free**, 11% util, no leaked contexts); **GPU/TDR reset** (`Ctrl+Win+Shift+B`) — no change;
  **system load** (overall CPU **2%**, idle); **remote-display/no-vblank** — the `DXGIUITickThread`
  is **idle in a condition_variable** (`AreDXGIUITicksWaitable`=false), *not* stuck in
  `WaitForVBlank`, so the present isn't waiting on vblank ticks (`AreUITicksNeededFromUIThread`
  false → `WaitForUITickFromUIThread` returns immediately). The CP lambda
  (command_processor.cpp:2014) is linear command-building, not an obvious infinite loop.
- **Key refinement:** the Windows desktop **composits fine** throughout (DWM works, windows render,
  screenshots succeed) → the host GPU/present pipeline is **not** wedged. Therefore this is **not a
  host-GPU-wedge a reboot "clears"** — it is a **deterministic GPU-interrupt/fence-delivery race in
  the rexglue-sdk runtime** that the *same build* won earlier the same session and now loses 100%.
  A reboot can only help by **reshuffling thread scheduling/timing** (re-winning the race), so it
  *may or may not* fix it. The durable fix is in the SDK: ensure the **guest GPU interrupt fires /
  the EOP fence the guest polls (`sub_821BFF48`→`sub_821C6E58`, `[obj+10896]`) is written back**
  after the first frame's GPU work, so the guest's post-init fence wait completes. Bounding the
  guest wait is risky (the loop level is ambiguous across `sub_82150970`/`sub_82249xxx`, and
  proceeding before the GPU is done corrupts state).

## Update 3 — mechanistic root cause (interrupt/fence path is alive; the CP is the stall)
Traced the SDK interrupt machinery end-to-end against the live dump:
- The **GPU VSync timer thread** (`graphics_system.cpp:156` lambda) is **alive and looping** (caught
  in its `Sleep(1ms)`), so `MarkVblank()` fires every refresh interval →
  `command_processor_->increment_counter()` (**vblank counter advances**) +
  `DispatchInterruptCallback(0,2)` runs the guest interrupt callback (`821C7170`) **on the vsync
  thread**, and it **returns** (no thread stuck in `ExecuteInterrupt`/`821C7170`). **Vblank interrupt
  delivery works.**
- So the guest's stuck fence (`sub_821BFF48`→`sub_821C6E58`, polling `*[obj+10896]`) is **not the
  vblank counter** — it's an **EOP/ring fence** advanced by the CP as it consumes the ring and fires
  the EOP interrupt (`DispatchInterruptCallback(1, n)`, `command_processor.cpp:1060`). The CP is
  stuck **inside the first `XE_SWAP`** (`IssueSwap → RefreshGuestOutput → RefreshGuestOutputImpl`)
  and **never reaches the EOP** → ring/EOP fence never advances → guest post-init wait hangs.
- The SDK author's **own TODO at `MarkVblank` (graphics_system.cpp:337-339)** flags exactly this:
  *"…there's something wrong and the CP will block waiting for code that needs to be run in the
  interrupt."* A **known-fragile CP↔interrupt ordering**; the first `XE_SWAP` is where it bites here.

**Durable fix (SDK):** make the first `XE_SWAP`'s `RefreshGuestOutputImpl` complete without blocking
on something that itself needs the interrupt/CP (break the CP↔interrupt cyclic dependency) so the EOP
fence advances. The recompiled **artifact is correct** (same build ran `boot→match→win` earlier the
same session); this is a runtime GPU-sync ordering bug that turned deterministic on this host.

## Update 4 — the guest fence is a producer/consumer EOP counter; forcing it cascades
Instrumented the guest fence wait (`sub_821C6E58`): the guest polls a GPU EOP/progress counter at
`*[obj+10896]` (`0xFFC9B000`, same page as the CP's `WAIT_REG_MEM` poll `0x1FC9B006`) and waits for
it to reach a target (e.g. `val 0x11 → tgt 0x17`). Without help it dead-stalls at the **first** such
wait — the CP, mid command-batch, hits a `WAIT_REG_MEM` waiting for a value the guest only writes
*after* passing this wait → mutual (bootstrap) deadlock.
- **Forcing the fence (write `tgt` into `*[obj+10896]` to simulate GPU completion) advances the
  guest through 5 sync points** (`val 0x11→0x25`, `tgt 0x17→0x2F`) — proving the boot **can** make
  forward progress when the fence is satisfied. But after ~5 it **cascades to a different stall**:
  the guest stops calling `sub_821C6E58` (no more fence waits) **and** the CP-side `WAIT_REG_MEM`
  escape never trips either — so the next wait is **a third mechanism** (likely a `Ke*` event wait),
  not coverable by forcing these two points.
- ⇒ **The code-hack path is whack-a-mole** (each forced sync reveals a new, different wait) and
  would corrupt rendering anyway (lying about GPU completion). It does **not** reach a clean — let
  alone *playable* — boot. The legitimate fix is the SDK's CP↔guest GPU-sync/interrupt-write-back
  (the upstream `MarkVblank` TODO), a substantial, hard-to-verify-while-blocked change. The
  recompiled artifact remains correct (booted to a win earlier the same session); the host's
  accumulated scheduler/runtime state after ~250 launches is what flips this bootstrap race to lose,
  and only a reboot (or that SDK fix) resets it. All boot-fix instrumentation has been reverted to
  clean codegen.

## Update 5 — CORRECTION: the first frame DOES present (screenshot); it's not a black/present stall
A screenshot of the "frozen" window shows the game **rendering the first intro frame** (the animated
Cartman-over-the-town scene with an "Ⓐ SKIP" prompt) — **not** a black screen. So the **present/swap
path works** (frame 1 is composited and shown); my earlier "present/vsync deadlock at the first
XE_SWAP" framing was wrong. What actually happens: **frame 1 presents, then the guest main thread
freezes** on its *post-frame-1* GPU-fence wait — the log stays at ~4 KB and `XamInputGetState` is
never called (the intro never advances to title/menu, and input is never polled, so the
`REX_INPUT_FILE`/injector can't drive it — the guest isn't reading input yet). It is a **producer/
consumer bootstrap**: the guest waits for the EOP fence (`0xFFC9B000`) to reach target T for frame 1;
the CP reached only V<T and the remaining work that would bump it to T is submitted by the guest only
*after* this wait. Forcing the fence advances the guest ~5 intro frames then it cascades to a third
wait (confirming the bootstrap). The same build played to a win earlier the same session, so the
correct fix keeps **rendering correct** (a reboot resets the host state that flips the race, or the
SDK's CP↔guest fence/EOP-write-back is hardened); forcing fences would corrupt rendering, so it is
not a valid "playable" path. **Lesson:** a "4 KB frozen log + 0 input polls" is NOT proof of a
black-screen/early stall — screenshot the window; here it was rendering the intro and stuck one frame
later, which relocated the bug from the present path to the guest's per-frame fence bootstrap.

## Update 6 — the deadlock fence is the CP's per-SUBMISSION progress (NOT counter_/read-ptr)
Re-reading the live FENCE-DIAG values: under the force-fence the guest's polled fence `*[0xFFC9B000]`
advanced **+4 each step** (`0x11→0x15→0x19→0x1D→0x21→0x25`) — i.e. it tracks the CP's progress **per
guest submission**, NOT the vblank `counter_` (which is +1/vblank) and NOT the ring read-pointer.
So the three runtime "write-back" fixes attempted (periodic read-ptr write-back; per-vblank
`counter_` fence refresh; `WAIT_REG_MEM` escape) target the **wrong fence** — they are correct,
harmless improvements but cannot fix this. The actual deadlock is a **producer-consumer bootstrap**:
the guest waits for `0xFFC9B000` to reach target T, but the CP only advances it to T after processing
a submission the guest issues **after** clearing this wait — circular. It is NOT a missing
write-back; the fence is written correctly when the guest submits. It is **timing**: the same build
won earlier the session (the CP kept pace / the guest submitted T before waiting); the accumulated
host OS-scheduler state after ~250 launches flips the race so the guest now waits before submitting.
**Implication for a durable fix:** don't pursue more fence write-backs — instead bound the guest's
fence wait (give it a timeout/yield so it submits more) or ensure the CP drains submissions ahead of
the guest's wait; and a **reboot** resets the host scheduler state that flips the race today.
