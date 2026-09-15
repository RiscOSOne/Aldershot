# 3. The evidence

The research followed the order from chapter 2.

<!-- doccrate:keep-together:start -->
## Step 1: the registers at the fault

`*ShowRegs` and `*Where` were redirected to a file on HostFS, so the dump could be read
exactly rather than by OCR from a screenshot.

| Register | Value |
|:---|:---|
| R0, R1 | `&001DD000`, `&1A8` |
| R2, R4, R8–R11 | `&00B5E8B5` |
| R3, R5 | `&00B2E4B3`, `&0018D69C` |
| R12, R13 | `&FFFF12B0`, `&FA207FA8` |
| R14, R15 | `&FC038210`, `&FC0382B8` |
| Mode and PSR | SVC32, `&20000113` |
<!-- doccrate:keep-together:end -->

The dump was stored at `&20003110`, and `*Where` placed the PC at offset `&00017FB4` in
the module `UtilityModule`.

<!-- doccrate:keep-together:start -->
## What the registers say

The faulting context is in SVC32, inside the kernel. It is not WimpForth's own user-mode
code, and the PC is not in a filing-system module. FileSwitch's buffered `OS_File 10`
copy loop is the natural candidate.

The buffer registers point at `&00B5E8B5` and `&0018D69C` — deep inside a 32 MB slot and
far beyond the 160 KB image, which is the untouched region being walked when the abort
fired. The RMA address in the Wimp's box and the kernel PC in the dump are consistent
with the dump having been captured in the abort handler.
<!-- doccrate:keep-together:end -->

<!-- doccrate:keep-together:start -->
## Step 2: the RAMFS control

A 64 MB RAM disc was made with `ChangeDynamicArea -RamFsSize 64M`, and the same save sent
to it with `fsave RAM::RamDisc0.$.TESTSAVE`.

It **aborted identically**, and the job runner then hung on the modal error box. A
`*ShowRegs` after the box was dismissed printed a dump identical to the HostFS one: the
same stored-context address, `&20003110`, and the same registers.
<!-- doccrate:keep-together:end -->

<!-- doccrate:keep-together:start -->
## What the RAMFS control does and does not prove

Two caveats are kept on record. The RAMFS error box was read by OCR prompted with the
address it was expected to show, and the identical dump may be a stale stored context
rather than a fresh one. But the runner hanging after the RAMFS save is hard evidence that
the RAMFS save killed the desktop too.

So the fault is **not HostFS-specific**. The mechanism the second opinion described — a
save spanning pages that are not yet mapped, faulting during an SVC-mode copy — survives.
On this evidence the faulting copy is FileSwitch's own buffered save, upstream of any
filing system.
<!-- doccrate:keep-together:end -->

<!-- doccrate:keep-together:start -->
## The slot-size control

The cheapest discriminator was run next, ahead of its turn.

| Launch | WimpSlot | `fsave TESTSAVE` |
|:---|:---|:---|
| through the application's own `!RUN` | `-max 2m -min 2m` | **succeeds**: 159,916 bytes on the host, runner alive |
| through `WimpTask` | the default, large | **aborts** |
<!-- doccrate:keep-together:end -->

The crash depends on the size of the slot, exactly as the second opinion's reading of the
two probes predicted. In a big slot, the range handed to `OS_File` — or the layout
WimpForth derives from its RAM limit — extends over pages the task has never touched.

<!-- doccrate:keep-together:start -->
## The workaround: touch the pages first

If the problem is pages that have never been touched, touching them first from user mode
should fix it, because a user-mode read of an unmapped page is an ordinary page fault the
demand pager serves.

#### Reading one byte of every page before the save

```forth
: touchit 32768 here over - bounds do i c@ drop 4096 +loop ;
touchit fsave TESTSAVE2      \ in the SAME big-slot configuration
```
<!-- doccrate:keep-together:end -->

`TESTSAVE2`, 160,000 bytes, saved cleanly, and the machine survived. Once every page in
the range has been read from Forth, the SVC-mode copy has nothing left to fault on. The
fix is now folded into UTILS as `"fsave` calling a `touch-range` word (fork commit
`<redacted>`).
