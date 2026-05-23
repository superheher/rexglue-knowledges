# Patching — mid-asm hooks, function overrides, and patches

Two complementary mechanisms let you change behaviour **without editing generated
code** (which is regenerated and must stay reproducible). Both are driven from
config + your `src/`, so they survive re-codegen. XenonRecomp documents the exact
forms; rexglue offers equivalents.

## Function overrides (replace a whole function)

The recompiler defines functions so they can be **weak-aliased** and overridden,
while keeping access to the original implementation:

```cpp
PPC_FUNC_IMPL(__imp__sub_XXXXXXXX);     // the original, recompiled body
PPC_FUNC(sub_XXXXXXXX)                   // your override (weak symbol replaced)
{
    // pre-work, or skip entirely…
    __imp__sub_XXXXXXXX(ctx, base);      // optionally call the original
    // post-work…
}
```

**Use when** you want to replace an entire guest routine with host logic:
- stub/replace networking, sign-in, or storage routines;
- swap a broken/expensive engine function for a host version;
- intercept asset I/O, save/load, or timing at function granularity.

## Mid-asm hooks (inject at one instruction)

A **host callback placed at a specific guest instruction address** (it does *not*
overwrite the instruction). Declared in config; you implement the function in
`src/` and the linker resolves it:

```toml
[[midasm_hook]]
name = "IndexBufferLengthMidAsmHook"
address = 0x82E26244
registers = ["r3"]          # which registers to pass in
# optional control:
# after_instruction = true  # place after, not before
# return = true             # return from the host function immediately after
# return_on_true / return_on_false = true
# jump_address / jump_address_on_true / jump_address_on_false = 0x...
```

```cpp
void IndexBufferLengthMidAsmHook(PPCRegister& r3) { /* read/modify r3 … */ }
```

Control options let the hook **return early** or **jump** within the same
function, conditionally on the hook's bool result. (Mutually exclusive
combinations are rejected with warnings — e.g. you can't both `return` and
`jump_address`.)

**Use when** you need a surgical change *inside* a function:
- patch a single value (clamp a length, fix an index);
- skip a problematic loop/branch;
- instrument (log a variable at a hot spot);
- inject host behaviour at an exact point without owning the whole function.

## Choosing between them

| Need | Use |
|---|---|
| Replace/repurpose an entire routine | **function override** |
| Tweak one value / skip a branch / instrument mid-function | **mid-asm hook** |
| Change data, not code | **data patch** (write guest memory at init) |
| Fix a structural translation issue (boundary, jump table, EH) | **config**, not a hook (`50`) |

## Best practices

- **Never hand-edit generated files.** Everything goes through config + `src/`
  so a re-codegen doesn't wipe your work.
- **Name hooks meaningfully** and keep a **registry** (address, purpose, date,
  related boot-log entry). Reuse a name to apply one implementation at multiple
  addresses.
- **Keep hooks minimal** and well-commented; a hook is a liability tied to an
  address that can move if the XEX/TU changes.
- **Re-validate after** bumping the toolchain, applying a Title Update, or
  toggling optimizations — addresses and codegen can shift.
- Cross-link each override/hook to the symptom it fixes in
  `titles/<id>/30-boot-log.md`.
