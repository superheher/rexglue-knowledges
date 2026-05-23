# Xbox 360 Static Recompilation — Knowledge Base

A growing, **reusable** body of knowledge for turning Xbox 360 games into native
PC builds by **static recompilation** (ahead-of-time machine-code → C++), not
emulation. It is written to be useful **across titles** and to be **published**
so other people — and other Claude Code / agent sessions — can build on it.

The running case study is *South Park: Let's Go Tower Defense Play!* (a small
XBLA title chosen as an onboarding pilot), but the `general/` material is
deliberately title-agnostic.

> Scope of this repo: **method, measurements, and references only.** No
> copyrighted game code or assets. Format/architecture facts are public
> knowledge (Xenia, Free60, and the upstream tools' own docs); empirical
> specifics are validated hands-on during bring-up and cited as such.

## How to use this KB

- **Starting a new 360 port?** Read `general/00` → `general/40`, then follow
  `general/90-new-title-onboarding-playbook.md` and copy `templates/`.
- **Debugging a specific failure?** Jump to `general/95-pitfalls-and-patterns.md`
  and the relevant subsystem doc (`50` CPU, `60` GPU, `70`/`75` runtime).
- **Choosing tools?** `general/30-toolchains.md`.
- **An agent consuming this:** treat `general/` as reference; treat
  `titles/<id>/` as the live state of one port; record new findings as you go.

## Layout

```
general/     transferable knowledge (the publishable core)
titles/      per-title case studies and live findings
templates/   copy-paste starting points for a new title
```

## Index

Legend: ✅ written · ✍️ skeleton (expand on contact) · ⏳ filled during bring-up

### `general/` — reusable
| Doc | Status | Topic |
|-----|:--:|-------|
| `00-static-recompilation-overview.md` | ✅ | What recomp is, why/when it beats emulation, the landscape & precedents, feasibility heuristics |
| `10-xbox360-hardware.md` | ✅ | Xenon CPU (PPC, VMX128, in-order, big-endian), Xenos GPU, memory map, EDRAM |
| `20-xex-format.md` | ✅ | XEX2 structure: headers, sections, base address, compression (LZX), encryption (AES), imports, `.pdata` |
| `25-containers-and-extraction.md` | ✅ | STFS (CON/LIVE/PIRS), GOD, ISO/GDFX; how to get `default.xex` + assets |
| `30-toolchains.md` | ✅ | XenonRecomp vs XenosRecomp vs rexglue-sdk vs rexdex; how to choose |
| `40-recompilation-pipeline.md` | ✅ | The end-to-end pipeline and where each tool fits |
| `50-cpu-recompilation.md` | ✅ | Jump tables, function boundaries, register save/restore, longjmp/setjmp, exceptions, endianness, FP/denormals, optimizations |
| `60-gpu-shader-translation.md` | ✅ | Xenos microcode → HLSL/SPIR-V, vertex fetch & formats, samplers, specialization, control flow |
| `70-runtime-kernel-and-xam.md` | ✅ | Implementing `xboxkrnl`/`xam`: memory, threads, TLS, sync, time, the import-shim methodology |
| `75-runtime-graphics-audio-input-io.md` | ✅ | Command processor, audio (XMA), input (XInput), filesystem/VFS, saves |
| `80-patching-hooks-overrides.md` | ✅ | Mid-asm hooks, function overrides, patch patterns and when to use each |
| `90-new-title-onboarding-playbook.md` | ✅ | **The reusable step-by-step checklist/decision tree for any new title** |
| `95-pitfalls-and-patterns.md` | ✅ | A catalog of recurring problems and their fixes |
| `98-glossary.md` | ✅ | Terminology |
| `99-references.md` | ✅ | Upstream docs, emulator/RE sources, articles, prior recomp projects |

### `titles/south-park-lgtdp/` — case study
| Doc | Status | Topic |
|-----|:--:|-------|
| `00-feasibility.md` | ✅ | Graded feasibility verdict + risk register for this title |
| `10-dump-analysis.md` | ✅ | Container layout, XEX/import findings, extraction plan |
| `20-imports-backlog.md` | ⏳ | Concrete kernel/XAM imports used: done/todo |
| `30-boot-log.md` | ⏳ | Boot bring-up journal (crash → cause → fix) |
| `40-render-notes.md` | ⏳ | Shader/render correctness findings |

### `templates/`
| Doc | Status | Topic |
|-----|:--:|-------|
| `title-feasibility.md` | ✅ | New-title feasibility report template |
| `title-onboarding-checklist.md` | ✅ | New-title onboarding checklist |
| `annotated-config.toml.md` | ✅ | Annotated recompiler config to copy & fill |

## Contributing & publishing

See `CONTRIBUTING.md`. The intent is to keep `general/` clean and citable so the
folder can be published as a standalone reference. Everything here is English-only.
