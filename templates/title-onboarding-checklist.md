# Onboarding checklist — <GAME TITLE> (`<TitleID>`)

Copy into `titles/<id>/` and tick off. Deep dives: `general/90-…playbook.md`.

## A. Project setup
- [ ] Super-repo created; toolchain submodules added (rexglue + XenonRecomp + XenosRecomp)
- [ ] Knowledge base added as a submodule
- [ ] Port repo scaffolded (`config/ tools/ src/ docs/ generated/`)
- [ ] `.gitignore` excludes dump, `private/`, `extracted/`, `generated/`, `*.xex*`

## B. Extract (Phase 1)
- [ ] Container identified (STFS CON/LIVE/PIRS · GOD · ISO/GDFX); Title ID noted
- [ ] `default.xex` extracted → `private/`; first 4 bytes `XEX2` verified
- [ ] Asset tree extracted → content root
- [ ] Title Update(s)/DLC found & classified; `.xexp` kept

## C. Recon & go/no-go (Phase 1)
- [ ] base · entry · image size
- [ ] encryption · compression
- [ ] imports (libraries) · distinct ordinal count
- [ ] `.pdata` present? function count
- [ ] middleware/static-lib fingerprints
- [ ] feasibility scored; scope decided; `00-feasibility.md` written
- [ ] **GO / NO-GO**: ____

## D. Toolchain (Phase 0)
- [ ] Clang 20+ · CMake 3.25+ · Ninja · pwsh 7 installed
- [ ] `git submodule update --init --recursive`
- [ ] rexglue-sdk built/installed; CLI runs

## E. Scaffold & codegen (Phase 2)
- [ ] `rexglue init` done; `file_path` → `private/default.xex` (+ patch)
- [ ] register save/restore addresses set
- [ ] longjmp/setjmp addresses set (if used)
- [ ] switch tables generated + reconciled
- [ ] explicit `functions` for jump-table routines
- [ ] `invalid_instructions` skips (EH/padding)
- [ ] codegen completes (`--force` as needed)
- [ ] **GATE: app links** ✅

## F. Boot bring-up (Phase 3)
- [ ] memory/heap · TLS · threads · time up
- [ ] VFS mounts asset tree (file opens succeed)
- [ ] missing imports stubbed/implemented as hit
- [ ] structural issues fixed (boundaries/jump tables/longjmp/EH)
- [ ] command processor presents a cleared frame, then a draw
- [ ] **GATE: first frame / menu shows** ✅

## G. Render (Phase 4)
- [ ] shaders correct (vertex formats, semantics, instancing, R11G11B10, alpha test)
- [ ] render state (blend/depth/raster, RT formats, EDRAM resolve, gamma)
- [ ] textures/samplers (formats, swizzles, filtering, cube)
- [ ] **GATE: menus + in-game look correct** ✅

## H. Playable (Phase 5)
- [ ] input (XInput → controller(s) + keyboard)
- [ ] audio (XMA → mixer → host out)
- [ ] save/continue across relaunch
- [ ] timing/pacing stable
- [ ] **GATE: boot → menu → match → win/lose → save → continue** ✅

## I. Polish (Phase 6)
- [ ] codegen optimizations enabled incrementally + re-verified
- [ ] perf pass; CVars/remap/resolution options
- [ ] Release packaged; `docs/RUN.md` written
- [ ] online/leaderboards/achievements stubbed to offline

## Records kept current
- [ ] `20-imports-backlog.md` · [ ] `30-boot-log.md` · [ ] `40-render-notes.md`
- [ ] general findings promoted to `general/95-pitfalls-and-patterns.md`
