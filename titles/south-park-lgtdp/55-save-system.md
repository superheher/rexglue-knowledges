# The save system — full architecture + why a blind run never triggers it (2026-05-24)

This is the deep characterization of the title's progress-save path, instrumented and
verified live. **Headline: the save subsystem is fully implemented and correctly wired;
it just never *requests* a save in any flow that can be driven blind (tutorial win →
continue, options, exit-to-menu). The save is request-queued + async, and the request is
only enqueued by a game progress/settings event that blind navigation doesn't reach.**

## The chain (endpoint → trigger), all source-verified in the codegen
| Function | Role |
|---|---|
| `sub_824485F8` | **endpoint** — calls `XamContentCreateEx` (the actual save); validates the content struct or bails error 87 |
| `sub_82448698` | thin wrapper over `sub_824485F8` (recomp.30:28153) |
| `sub_82129AE8` | serialize + save (content-type 3) |
| `sub_82129958` | **save-decision** — `(r3=index, r4, r5, r6=&entry)`; writes the entry header then serializes. Called **per queued entry** |
| `sub_82129730` | **async save state machine / pump** — global state at **`0x82919B40`** (states 0→1→2→3→4). Iterates a **queue of 308-byte save-entries** (`[statebase+44]`, count in `r28`) and calls `sub_82129958` for each |
| `sub_82151170` | **flush-if-dirty** — `if ([saveobj+25]==0) return;` else pumps `sub_82129730`. `saveobj = 0x827F4D8C` (fixed global). Called **~35×/s (17 638×/run)** |
| `sub_8215E2E0` | **unconditional save-event** — pumps `sub_82129730` with no dirty gate |
| `sub_8215D348` | **save-handler** that contains all **4** call sites of `sub_8215E2E0`. **No direct callers — invoked indirectly** (registered callback / save worker; only appears in the function table + `SetFunction`) |

So there are two ways the pump (`sub_82129730`) ever runs: (a) the **dirty flag** `[0x827F4D8C+25]` is set, so the per-frame `sub_82151170` stops skipping; or (b) the save-handler `sub_8215D348` runs and calls `sub_8215E2E0`. **Neither happens in a blind run.**

## Live evidence (instrumented, this build)
- `[SAVE2]` on `sub_82151170`: fires **17 638×/run**, but **`dirty[+25]=0` every single time** → the flush always early-returns. The save-data object is **never marked dirty**.
- `[SAVE-DIAG]` on the save-decision `sub_82129958`: **0 fires** in every driven flow (win→continue, HELP & OPTIONS, EXIT GAME) → the entry queue is always empty when pumped.
- `[SAVE3]` on the save-handler `sub_8215D348`: **fires once at boot** with non-guest pointers (`this=0x02C6…`, an init/registration artifact) and **never again** with a real request.
- Forcing the dirty flag true (override `[+25]` for a range of frames) **does** make `sub_82151170` pump `sub_82129730`, but it **still never reaches `sub_82129958`** — because the entry queue is empty + the async state needs a genuine queued request, not just a pump. So forcing the *gate* is the wrong lever; the missing thing is the **enqueue**.
- The write path itself is proven good: the shader cache writes to `user_data_root` fine, and `XamContentCreate*`/dummy storage devices are implemented (see [[50-menu-input-and-lobby]]).

## What this means for Condition A's "save/continue"
The recomp's save is **not broken** — it is **un-triggered**. To make it fire you must reach
the game event that **enqueues a save-entry** (or sets `[0x827F4D8C+25]`). Candidates, none
reachable by the blind injector path used so far:
- completing a **non-tutorial** stage (Stan's House is the tutorial; later stages are locked
  behind progress that itself needs the first save — a bootstrap the tutorial-complete save
  is presumably meant to write);
- a **settings commit** in HELP & OPTIONS (changing a value, then backing out) — navigated to
  but no value was actually changed/committed;
- exit-to-dashboard with unsaved progress.

**Next concrete leads** (for a human play-test or a focused next session):
1. Find the **enqueue**: `sub_8229BFE8` (recomp.13, heavy 308-byte-stride use — the save-entry
   array manager) and the `sub_8229Bxxx` family (`sub_8229BBE8` init, `sub_8229BD90`). Find who
   adds entries to `[0x82919B40+44]` and increments the count.
2. Find what **indirectly invokes** `sub_8215D348` (it's a registered worker/callback) and what
   posts requests to it — that's the explicit "save now" path.
3. Or simply **play to a real save point**: the `[SAVE-DIAG]`/`[SAVE3]` instrumentation is left
   in this build, so a human playthrough will print `[SAVE-DIAG] … (SAVING)` the instant the
   game saves — pinpointing the trigger with zero extra work.

## Reusable lesson (promoted to general/95)
Recompiled console saves are often **async + request-queued state machines**, not synchronous
`fwrite`s. A per-frame "flush if dirty" that you see firing thousands of times with the dirty
flag always 0 is **not** the trigger — it's the *pump*. Trace to the **enqueue** (who marks
dirty / who posts the request), and don't be fooled into "forcing" the gate: that pumps an
empty queue. Leave the endpoint (`XamContent*`) instrumented so a real playthrough self-reports.
