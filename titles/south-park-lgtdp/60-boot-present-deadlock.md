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
