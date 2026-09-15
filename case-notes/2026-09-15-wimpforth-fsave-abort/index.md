# WimpForth: an fsave that aborts the machine

**A case note from the RISC OS Pi 4 emulator. WimpForth metacompiles its own kernel on
the emulator without trouble, but saving the finished application image kills the
desktop with "abort on data transfer" — in a large WimpSlot, and never in a small one.**

<!-- doccrate:keep-together:start -->
## The case at a glance

| | |
|:---|:---|
| **Status** | **Unsolved** at the operating-system level. A workaround is in place and validated. |
| **Filed** | 15 September 2026 |
| **Reproduces** | Every time WimpForth saves its image while running in a large WimpSlot |
| **Machine** | RISC OS 5.30 on QEMU's emulated Raspberry Pi 4, booting from HostFS 2.03 |
| **Affects** | WimpForth's whole-image save, and plausibly any program that saves a large, partly untouched memory range with `OS_File 10` |
<!-- doccrate:keep-together:end -->

<!-- doccrate:keep-together:start -->
## The short version

WimpForth saves its running image by handing `OS_File 10` the whole range from `&8000`
to the top of its dictionary. In a large WimpSlot much of that range has never been
touched, so its pages are not yet mapped. The save is copied in SVC mode, where touching
an unmapped page is a data abort rather than a page fault the kernel can quietly serve,
and the machine goes down.

Reading one byte of every page from user mode before the save — which makes the demand
pager map them — lets the same save complete in the same slot. That is the workaround.
What is still not known is exactly which copy loop faults, and why the range reaches
past mapped memory in the first place.
<!-- doccrate:keep-together:end -->

<!-- doccrate:keep-together:start -->
## Where the source is

WimpForth's upstream is [alban-read/forth](https://github.com/alban-read/forth). The emulator, its HostFS 2.03
module and the `vmchannel` device it talks to are in
[albanread/RISCOSQEMUA72](https://github.com/albanread/RISCOSQEMUA72), branch
`riscos-pi4`. Details of the private test farm the case was worked on are shown as
`<redacted>`.
<!-- doccrate:keep-together:end -->

<!-- doccrate:keep-together:start -->
## Chapters

| Chapter | What it covers |
|:---|:---|
| [1. The case as filed](01-the-case.md) | what works, what dies, the save path, and the first theory |
| [2. A second opinion](02-a-second-opinion.md) | the right HostFS, a path that can abort, a lost digit, and a plan |
| [3. The evidence](03-the-evidence.md) | registers at the fault, the RAMFS control, the slot-size control, the workaround |
| [4. A second bug on the way](04-a-second-bug.md) | a one-character delimiter, and why guest sources stay Latin-1 |
| [5. Conclusions](05-conclusions.md) | what this says about WimpForth, and what is still not established |
| [6. Status and open items](06-open-items.md) | the order the remaining questions would be picked up in |
<!-- doccrate:keep-together:end -->

<!-- doccrate:keep-together:start -->
## How to read the diagrams

The diagrams colour each box's outline by who owns that code.

| Outline | Meaning |
|:---|:---|
| purple | the application: WimpForth |
| grey | RISC OS itself |
| navy | a filing system: HostFS or RAMFS |
| teal | the host, or the emulator's device |
| red | where the fault happens |
<!-- doccrate:keep-together:end -->
