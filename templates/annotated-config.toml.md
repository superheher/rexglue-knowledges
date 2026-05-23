# Annotated recompiler config (copy & fill)

The schema below follows **XenonRecomp**'s documented config (the clearest spec);
**rexglue** exposes equivalent settings in its `*_config.toml`. Replace every
`0x…` with values from your XEX recon. All paths are relative to the TOML file.

```toml
[main]
# The executable to recompile. If a Title Update exists, give the base XEX and
# the patch; the tool produces and reuses the patched XEX.
file_path          = "../private/default.xex"
# patch_file_path  = "../private/default.xexp"        # only if a TU exists
# patched_file_path= "../private/default_patched.xex" # auto-created from the patch
out_directory_path = "../generated/ppc"               # must exist before running
switch_table_file_path = "switch_tables.toml"         # from XenonAnalyse/scanners

# --- Optimizations -----------------------------------------------------------
# Leave ALL false until you have a stable, correct boot. Enable one at a time
# and re-verify; they assume a clean ABI / no exceptions. (general/50)
skip_lr             = false
skip_msr            = false
ctr_as_local        = false
xer_as_local        = false
reserved_as_local   = false
cr_as_local         = false
non_argument_as_local = false
non_volatile_as_local = false

# --- Register save/restore helper addresses ----------------------------------
# Find by byte pattern (general/50) or symbols. Wrong values corrupt regs/stack.
restgprlr_14_address = 0x________   # pattern e9 c1 ff 68
savegprlr_14_address = 0x________   # pattern f9 c1 ff 68
restfpr_14_address   = 0x________   # pattern c9 cc ff 70
savefpr_14_address   = 0x________   # pattern d9 cc ff 70
restvmx_14_address   = 0x________   # pattern 39 60 fe e0 7d cb 60 ce
savevmx_14_address   = 0x________   # pattern 39 60 fe e0 7d cb 61 ce
restvmx_64_address   = 0x________   # pattern 39 60 fc 00 10 0b 60 cb
savevmx_64_address   = 0x________   # pattern 39 60 fc 00 10 0b 61 cb

# --- longjmp / setjmp (omit if unused) ---------------------------------------
# longjmp near RtlUnwind references; setjmp often just after it.
# longjmp_address = 0x________
# setjmp_address  = 0x________

# --- Explicit function boundaries --------------------------------------------
# For functions the analyzer misses (commonly jump-table-laden ones).
functions = [
  # { address = 0x________, size = 0x__ },
]

# --- Invalid-instruction skips -----------------------------------------------
# Skip data that isn't code: exception handlers, padding, end-of-.text markers.
invalid_instructions = [
  { data = 0x00000000, size = 4 },   # padding
  # { data = 0x________, size = 8 },  # C++ frame handler
  # { data = 0x________, size = 8 },  # C-specific frame handler
  # { data = 0x________, size = 44 }, # end of .text
]

# --- Mid-asm hooks (one block per hook) --------------------------------------
# Host callback injected at an instruction address; implement in src/. (general/80)
# [[midasm_hook]]
# name      = "SomeMidAsmHook"
# address   = 0x________
# registers = ["r3", "r4"]
# after_instruction = false
# return    = false
# return_on_true = false
# return_on_false = false
# jump_address = 0x________
# jump_address_on_true  = 0x________
# jump_address_on_false = 0x________
```

Companion `switch_tables.toml` (generated, then hand-corrected) defines each jump
table so codegen emits real `switch` statements. See `general/50` for how to
detect tables (`mtctr`/`bctr`) and `general/30` for how to run the analyzer.
