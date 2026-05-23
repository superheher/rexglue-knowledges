# The end-to-end recompilation pipeline

How the pieces fit, in order, and which tool owns each step. Tool-specific
commands are in `30-toolchains.md`; the per-title checklist is in
`90-new-title-onboarding-playbook.md`.

```
  ┌──────────────┐   extract    ┌───────────┐   recon     ┌──────────────────┐
  │  dump (STFS/ │ ───────────▶ │ default.  │ ──────────▶ │ recompiler config│
  │  GOD/ISO)    │   + assets   │ xex (+TU) │  imports/   │ (TOML: boundaries│
  └──────────────┘              └───────────┘  base/pdata │  switch tables,  │
        │  assets                     │                    │  save/restore…)  │
        ▼                             ▼                    └────────┬─────────┘
  ┌──────────────┐            ┌───────────────┐                    │ codegen
  │ content root │            │ shader blobs  │                    ▼
  │ (VFS mount)  │            │ (in XEX/assets)│            ┌──────────────┐
  └──────┬───────┘            └──────┬────────┘            │ generated C++ │
         │                           │ shader recomp        │ (PPC→C++)     │
         │                           ▼                       └──────┬───────┘
         │                    ┌───────────────┐                     │
         │                    │ HLSL→DXIL/    │                     │
         │                    │ SPIR-V cache  │                     │
         │                    └──────┬────────┘                     │
         │                           │                              │
         ▼                           ▼                              ▼
  ┌─────────────────────────────────────────────────────────────────────┐
  │ host runtime (kernel/XAM, command processor, audio, input, VFS) +    │
  │ generated CPU code + shaders  ── compile/link (Clang) ──▶ native EXE  │
  └─────────────────────────────────────────────────────────────────────┘
                                     │ run → debug → fix → re-gen ↺
                                     ▼
                              playable build
```

## Steps

1. **Extract** `default.xex` (+ any `.xexp` TU) and the **asset tree** from the
   dump (`25-containers-and-extraction.md`).
2. **Recon** the XEX: base address, entry, imports/ordinals, `.pdata`,
   compression/encryption, TU presence (`20-xex-format.md`). Record it.
3. **Configure** the recompiler: point it at the (patched) XEX; declare register
   save/restore addresses, `longjmp`/`setjmp`, explicit function boundaries,
   invalid-instruction skips, and **jump/switch tables** (`50-cpu-recompilation.md`).
4. **Analyze** for jump tables (XenonAnalyse / rexglue scanners) and reconcile
   with the config; iterate.
5. **CPU codegen**: translate PPC → C++ (use `--force` to push past unresolved
   calls and converge). Output is many `.cpp`/headers + a `ppc_config.h`.
6. **Shader recomp**: translate Xenos shaders → HLSL → DXIL/SPIR-V, either as a
   prebuilt cache (XenosRecomp) or via the runtime's translation path (rexglue)
   (`60-gpu-shader-translation.md`).
7. **Runtime**: stand up the host — kernel/XAM imports, command processor,
   audio, input, VFS mounting the asset tree (`70`/`75`).
8. **Compile/link** generated code + shaders + runtime with **Clang** into a
   native executable.
9. **Bring-up loop**: run → triage crash/visual/audio bug → add a shim, hook,
   override, or config fix → (re-gen if needed) → repeat. This loop is where the
   time goes; record findings in the title's case-study docs.
10. **Optimize & package** once stable (`50` optimizations; `95` pitfalls).

## Artifacts & where they live

| Artifact | Committed? | Location |
|---|---|---|
| Recompiler config (TOML), switch tables | **yes** (config, not content) | `<port>/config/` |
| Extraction tooling | **yes** | `<port>/tools/` |
| Host app, shims, hooks, overrides | **yes** | `<port>/src/` |
| Extracted `default.xex`, assets | **no** (git-ignored) | `<port>/private/`, content root |
| Generated C++ / shader cache | **no** (reproducible) | `<port>/generated/` |
| Findings, logs, screenshots | **yes** | `knowledge-base/titles/<id>/` |

## The golden rule

**Get to a booting frame as fast as possible**, even ugly, even with everything
stubbed. Until it runs you are guessing; once it runs each fix is concrete and
observable. Prefer stubs that let execution continue over perfect
implementations that delay the first run.
