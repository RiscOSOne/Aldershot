# 9. The unpublished half

This last chapter collects what the published repositories do not contain, what that
means for anyone trying to reproduce the work, and an honest ledger: the project's stated
gates against what later commits show, the risks found by reading the code, and the
licensing points a reader should know.

## Relocatable modules

RISC OS extends itself with **relocatable modules**: code loaded into the module area, the
RMA, and entered in supervisor mode through a header of entry points. HostFS and GVFill are
modules. A language that can build modules can extend the OS itself.

### Advertised, but not published

The ROSCC README lists a second link mode, `roscc link --module`, linking at base 0 into a
relocatable module with its header taken from a `.module` section. **The published source
has no such option.** Given `--module`, the published linker treats it as a file name.

roscc's own review of 10 September names the files the module path lives in —
`src/module.rs`, a module runtime, a header generator, a build script, a live test, and
tests — and its first item says they are **untracked**. It calls the module path the best
work in the crate, and asks for it to be committed before anything touches it. The review
also mentions a debugging switch that is absent from the published source. **INFERRED:**
the roscc used for the demos and for the first HostFS module was newer than the published
source.

### The module ABI, reconstructed

The QEMU fork's first HostFS module was built with roscc on 10 September, and that build
script is public. With the review, it shows the module ABI:

```bash
# -O0 and -mno-movt matter: they force literal-pool addressing of
# statics (SBREL32), the one static-base relocation the module linker
# applies.  MOVW/MOVT pairs (BREL) come out at -O1, or on any CPU with
# Thumb-2 extensions unless movt is disabled
CC_FLAGS="--target=$TRIPLE -mcpu=$CPU -mfloat-abi=soft -ffreestanding -nostdlib -fno-builtin -fropi -frwpi -O0 -mno-movt $MOD_ARENAS"
...
"$ROSCC" link --module -o "$OUT/HostFS,ffa" \
    "$OUT/module_head.o" "$OUT/entries.o" "$OUT/hostfs.o" \
    "$OUT/rostrt.o" "$OUT/swis_os.o" "$OUT/modrt.o" \
    "$OUT/aeabi.o" "$OUT/atomics.o"
```

<!-- doccrate:keep-together:start -->

#### The pieces of the ABI

| Piece | How it works |
|:---|:---|
| code | `-fropi`: position-independent, reached PC-relative |
| statics | `-frwpi`: reached relative to a static base in `r9`, and claimed from the RMA at initialisation |
| addressing | `-O0 -mno-movt`, so statics use literal pools: the one static-base relocation the module linker applies |
| header and veneers | generated from a title, help text, init and final functions, and commands |
| runtime | built with `-DROSTRT_STATIC_HEAP`, because a module has no application slot |

<!-- doccrate:keep-together:end -->

The entry veneers are hand-written assembly in the same fork. Each one saves the caller's
registers as a block, loads the static base from the module's private word, calls C, and
turns a non-zero return into an error with the V flag set:

```asm
hostfs_fs_\name:
    stmfd   sp!, {r0-r9, lr}        @ the caller's registers, as a block
    mov     r0, sp                  @ -> that block
    ldr     r9, [r12]               @ static base from the private word
    ldr     r1, [r9, #-8]           @ the module workspace pointer
    bl      \csym
    cmp     r0, #0
    bne     91f
    ldmfd   sp!, {r0-r9, lr}        @ hand back whatever the handler wrote
    msr     cpsr_f, #0
    mov     pc, lr
91: mov     r11, r0                 @ park the error pointer out of the
    ldmfd   sp!, {r0-r9, lr}        @   unwind's reach
    mov     r0, r11
    msr     cpsr_f, #(1 << 28)
    mov     pc, lr
```

Two runtime rules exist only for modules, and both appear earlier in this document. Every
SWI shim clobbers `lr`, because a SWI in supervisor mode overwrites it (chapter 5). And
`GetOrCreateGlobal` must return the same block on every entry (chapter 4).

<!-- doccrate:keep-together:start -->

### Where modules stand

| Build | State |
|:---|:---|
| a Mojo module, StrongARM | **48 of 48 live checks passed** on RPCEmu, cold boot 8 s, driven through the guest portal |
| the same module, Pi 4 | **does not link**: with `-fropi -frwpi` for ARMv8, clang reaches statics through `MOVW`/`MOVT` pairs, relocation types 84, 85, 45 and 46, which the linker refuses |
| HostFS, built with roscc for the Pi 4 | built on 10 September with the `-O0 -mno-movt` workaround; HostFS later moved to a DDE build to run from ROM |

<!-- doccrate:keep-together:end -->

The review's first fix for the Pi 4 is to add the four relocation types, with the same
immediate packing as the existing `MOVW`/`MOVT` cases.

**INFERRED, from the public build script:** rebuilt today, it would pick up the runtime's
application-slot heap, because it does not pass `-DROSTRT_STATIC_HEAP`. And it assembles the
`SWP` atomics for the Cortex-A72, where ARMv8 no longer provides `SWP` in AArch32.

[[pagebreak]]

<!-- doccrate:keep-together:start -->

## What a clean clone can do

| Wanted | From public sources today |
|:---|:---|
| build roscc | yes, after pointing its LLVM configuration at a local install |
| build the runtime objects | yes: all 45 shim files compile with zero warnings, for both profiles |
| regenerate the bindings | **no**: the SWI database is private |
| build the ARM-enabled `mojo` | **no, as published**: the backend list lacks `ARM`; add it and rebuild, about an hour |

<!-- doccrate:keep-together:end -->

<!-- doccrate:keep-together:start -->

#### What a clean clone can do, continued

| Wanted | From public sources today |
|:---|:---|
| build `hello_world` | only with a locally ARM-enabled `mojo` |
| build Othello or Mandelbrot | **no**: they import the unpublished `demos/` |
| link a relocatable module | **no**: the module writer is unpublished |

<!-- doccrate:keep-together:end -->

<!-- doccrate:keep-together:start -->

## Gates, stated and actual

The fork's README has not changed since 11 September. Later commits have moved past it:

| Gate | README says | Later commits show |
|:---|:---|:---|
| R1: the ARM backend in LLVM | done | done, in a local build; not in the published configuration |
| R2: a RISC OS target in the driver; roscc as linker | open | still open; the link is a separate step |
| R3: the standard library's system layer on SWIs | first cut | the runtime surface of chapter 4 |

<!-- doccrate:keep-together:end -->

<!-- doccrate:keep-together:start -->

#### Gates, continued

| Gate | README says | Later commits show |
|:---|:---|:---|
| R4: the Wimp in the library, generated from the PRM | done for 126 SWIs | 440 of 584 SWIs reachable |
| R5: a Wimp window in Mojo, running | built, not yet run | windows run on RPCEmu, and on the Pi 4 |

<!-- doccrate:keep-together:end -->

The README and the port log also still describe VFPv4 and NEON for the A72 profile, and
call RPCEmu's fault trap a GDB stub.

<!-- doccrate:keep-together:start -->

## Risks found by reading the code

Each item is **INFERRED**, and not tested:

| Where | Risk |
|:---|:---|
| `KGEN_CompilerRT_GetOrCreateGlobal` | rostrt's signature differs from the upstream runtime's; on 32-bit ARM the arguments land in different registers (chapter 4); check with the real compiler |
| `crt0` and `os_exit` | an exit code is passed without the word `OS_Exit` needs before honouring it |
| `wimp.report_error` | copies the message from byte 1 of the error block, where RISC OS error text follows a 4-byte number |
| `wimp.simple_window` | never sets the horizontal scroll word, and the arena is no longer zeroed `.bss` |

<!-- doccrate:keep-together:end -->

<!-- doccrate:keep-together:start -->

#### Risks found by reading the code, continued

| Where | Risk |
|:---|:---|
| `wimp.poll` | the generated shim takes a third register the hand binding never passes |
| the linker | locals binding to same-named globals; GOT slot collisions; ignored `MOVW`/`MOVT` addends |
| `.gitignore` | its patterns miss the build products `pi4.sh` writes under `build/` and `demos/` |
| long strings | whether the alignment fix cured them is recorded both ways (chapter 4) |

<!-- doccrate:keep-together:end -->

<!-- doccrate:keep-together:start -->

## Licensing, in brief

| Item | Position |
|:---|:---|
| **ROSCC** | MIT since `707a35f`; Apache-2.0 before. The commit notes that Apache 2.0 carries an express patent grant and MIT does not. Its `NOTICE` records that LLVM is linked, not distributed, and that no RISC OS Open or Acorn source is reproduced |
| **MojoRISCOS** | Apache-2.0 with LLVM exceptions, inherited from upstream. Modular's per-file headers must be kept; no Modular binaries; nothing routed to Modular for support |
| **names** | Mojo, MAX and Modular are Modular's trademarks. The fork is not Modular's, and a build must not be presented as if it were. This document's title names the language the fork compiles, not an official product |

<!-- doccrate:keep-together:end -->

<!-- doccrate:keep-together:start -->

#### Licensing, continued

| Item | Position |
|:---|:---|
| **`max/`** | back in the published tree with the full history, and out of scope: the README gives a licensing reason. Its wording and the `NOTICE`'s differ on which licence governs it |
| **the SWI database** | derived from the PRMs, and kept private. Generated bindings cite page numbers, and quote nothing |
| **the QEMU fork and RPCEmu Instrument** | GPL-2.0-or-later; the short excerpts in this document are quoted with attribution |

<!-- doccrate:keep-together:end -->

## The review's order of work

roscc's review of 10 September ends with a priority order. It is still the shortest honest
description of what comes next:

1. commit the module work
2. add the `MOVW`/`MOVT` static-base and PC-relative relocations, so the Pi 4 module links
3. fix section alignment and duplicate-symbol detection
4. build a runner for the QEMU Pi 4 once HostFS can carry files — which it now can, and which
   the farm and `keys.py` already do by hand
5. generate SWI X-form variants, and handle UTF-8 on a Latin-1 console
6. tooling hygiene

To that list, this document would add the items the demos found: move the hand-written Wimp
fixes into the generator's appendix, so regeneration keeps them; publish `demos/`; and give
roscc archive member pulling, so the library can become the archive the Mandelbrot demo's
comment asks for.
