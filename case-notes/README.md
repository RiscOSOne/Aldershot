# Case notes

Problems worth writing down, met while running real software on RISC OS
on QEMU: what the program did, what was tried, what the evidence showed,
and what is still open. Each case reads chapter by chapter here on
GitHub, and each has a PDF of the whole thing.

| Filed | Case | Status | Read |
| --- | --- | --- | --- |
| 15 September 2026 | **WimpForth: an fsave that aborts the machine** — WimpForth metacompiles its own kernel without trouble, but saving the finished application image kills the desktop with "abort on data transfer" in a large WimpSlot, and never in a small one. | Open at the operating-system level; a workaround is in place | [Chapters](2026-09-15-wimpforth-fsave-abort/index.md) · [PDF](2026-09-15-wimpforth-fsave-abort/WimpForthFsaveAbort-Case.pdf) |

The newest case goes at the top.

Cases are written up from the working notes kept while they were
investigated. Anything private to the machines they were worked on, such
as local paths, test-farm details and unpublished commits, is shown as
`<redacted>`.

## Filing a case

Each case has its own folder, named for the day it was filed and what it
is about, such as `2026-09-15-wimpforth-fsave-abort`. Inside are an
`index.md` with the case at a glance and a table of chapters, one
Markdown file per chapter, and the PDF, which is exported from the same
files. Its row in the table above says where the case stands, and
changes when that does.

The emulator the cases ran on is
[albanread/RISCOSQEMUA72](https://github.com/albanread/RISCOSQEMUA72),
branch `riscos-pi4`. The [developer walkthroughs](../walkthroughs/README.md)
explain how it works.
