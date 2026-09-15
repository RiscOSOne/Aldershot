# 6. Status and open items

<!-- doccrate:keep-together:start -->
## Status

**Unsolved at the operating-system level. A workaround is in place and validated.**

The workaround lives in the WimpForth tree, as UTILS's `"fsave` calling `touch-range`
(fork commit `<redacted>`), and every rebuild from now on exercises it.
<!-- doccrate:keep-together:end -->

<!-- doccrate:keep-together:start -->
## Open items

In the order they would be picked up.

| # | Item |
|:---|:---|
| 1 | Capture the RAMFS error box without a primed prompt — a fresh screenshot, no expected address — and a `*ShowRegs` straight after a RAMFS-only abort, to settle RMA against kernel PC. |
| 2 | Log the real R4, R5 and slot size to a file from inside `"fsave`, just before the SWI: step 4 of the plan, not yet run. |
| 3 | Run a HostFS save under [`VMCH_TRACE`](https://github.com/albanread/RISCOSQEMUA72/blob/riscos-pi4/hw/misc/vmchannel.c): a `translate failed at va=` line would show exactly where the range stops being mapped. |
<!-- doccrate:keep-together:end -->

<!-- doccrate:keep-together:start -->
#### Open items, continued

| # | Item |
|:---|:---|
| 4 | Identify `UtilityModule` `+&17FB4` in a ROM with symbols. |
| 5 | Harden HostFS regardless: call `OS_ValidateAddress` in PutBytes and GetBytes before touching pages, so a dead virtual machine becomes an error the client can report. |
| 6 | Write up the WimpForth fix for upstream. Touching the range first is generic: it applies to any Forth that saves a whole image through `OS_File 10` on RISC OS 5 in a large slot. |
<!-- doccrate:keep-together:end -->

<!-- doccrate:keep-together:start -->
## The environment

| | |
|:---|:---|
| **Host** | a Mac Pro running macOS |
| **Emulator** | [RISCOSQEMUA72](https://github.com/albanread/RISCOSQEMUA72)'s `qemu-system-aarch64 -M raspi4b -cpu cortex-a72,aarch64=off` |
| **ROM** | stock RISC OS 5.30 with HostFS 2.03 spliced in (`<redacted>`); no network |
| **HostFS share** | `<redacted>`, file types carried as `,xxx` suffixes |
| **Driven by** | a private test farm, over QMP: run, type, screenshot (`<redacted>`) |
<!-- doccrate:keep-together:end -->

<!-- doccrate:keep-together:start -->
## Launching WimpForth

| Launch | Slot | Outcome |
|:---|:---|:---|
| `Dir HostFS:$.!WimpForth`, then `WimpTask FWIN32` | large | aborts on save |
| through `!RUN`, which sets `WimpSlot 2m` | 2 MB | saves |
<!-- doccrate:keep-together:end -->

<!-- doccrate:keep-together:start -->
## The source and the evidence

| What | Where |
|:---|:---|
| WimpForth, upstream | [alban-read/forth](https://github.com/alban-read/forth) |
| The WimpForth fork the case was worked on | `<redacted>` |
| HostFS 2.03 module source, `touch_pages` at line 651 | [`riscos-pi4/hostfs/dde/c/hostfs`](https://github.com/albanread/RISCOSQEMUA72/blob/riscos-pi4/riscos-pi4/hostfs/dde/c/hostfs#L651) |
| The `vmchannel` device | [`hw/misc/vmchannel.c`](https://github.com/albanread/RISCOSQEMUA72/blob/riscos-pi4/hw/misc/vmchannel.c) |
| Register dumps after the HostFS and RAMFS aborts; the saved `TESTSAVE`, `TESTSAVE2` and `FWIN`; screenshots of both abort boxes and the new `FWIN` running | `<redacted>` |
<!-- doccrate:keep-together:end -->
