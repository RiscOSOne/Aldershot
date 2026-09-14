# 8. Measured, limited, unfinished

This chapter gathers the measurements, then what is still open: first as the project
states it, then as found by reading the code. It ends with the question the project
raised about what could ever be shipped.

## SD card against HostFS

The boot design predicted HostFS would beat the emulated SD card substantially, and gave
the reason as crossings. Every register access to the emulated SD controller traps into
QEMU, and a 512-byte sector through a FIFO is about 128 of them, before FileCore's own
work. HostFS is one crossing per filing-system operation.


<!-- doccrate:keep-together:start -->

### The boot, timed

The measurement used the same ROM and the same tree, with the card's extra start-up file
removed so that both booted identical `!Boot` sequences. It was run on the Mac under TCG,
to a settled desktop:

| | SD card | HostFS |
|:---|:---|:---|
| wall clock, three runs | 16.3, 15.1, 15.8 s: **15.7 s** | 14.0, 14.5, 14.8 s: **14.4 s** |
| host crossings | 2,434,544 SD controller register accesses | 2,657 doorbells |
| of which data | 2,351,650 reads of the data port | 311 reads, whole |
| device commands | 827, including 730 multi-block reads | — |
| host time in the device | not measured | 0.103 s in all: 39 µs a request |

<!-- doccrate:keep-together:end -->


HostFS is faster by 1.3 seconds, 8%. The design record explains why nine hundred times
fewer crossings buy so little:

> the boot is bound by the emulated CPU, and under TCG an MMIO access is a function call,
> not an exit, so nine hundred times fewer crossings buy little. Under hardware
> virtualisation each of those 2.4 million accesses would be a VM exit, and the gap would
> be the design's argument; under TCG it is a footnote.


<!-- doccrate:keep-together:start -->

### Where the doorbells go

One boot from the share, counted from the trace:

| Operation | Doorbells |
|:---|---:|
| catalogue reads: *does this exist, and what is it?* | 1,894 |
| … of which misses, where the file was not there | 722 |
| data reads | 315 |
| opens and closes | 203 |
| directory listings | 32 |
| buffers faulted in and retried | 4 |

<!-- doccrate:keep-together:end -->


The boot design had warned that existence checks would run into the thousands, and they
do. A miss on a typed name scans the whole directory, because `notes` might be stored as
`notes,ffb`. An index was planned before any boot attempt. Measured instead, **the 722
missing lookups cost 33 ms of a 14.8-second boot**, so the index stays unbuilt until a
workload says otherwise.


<!-- doccrate:keep-together:start -->

### Four ways to lose the win

The boot design listed the mistakes that would give the advantage away, and the build
avoids all four:

| Mistake | HostFS today |
|:---|:---|
| a host `open`, `stat` and `close` per RISC OS operation | one request per FileSwitch call |
| `fsync` on every write | none |
| a coherence `stat` on every open | none; the host file system is the truth |
| a doorbell per catalogue entry | a whole page of entries per doorbell |

<!-- doccrate:keep-together:end -->


<!-- doccrate:keep-together:start -->

## Design against build

| Area | The design | Built |
|:---|:---|:---|
| translation | the wire speaks FileSwitch | a hybrid of version 1 frames and version 0 commands |
| addresses | logical; the host walks the MMU | the request block physical; buffers logical |
| directories | listings cached with a time-to-live | no cache; paged and sorted per request |
| errors | a registered error base | named errors, on unregistered numbers |
| security | `realpath` containment, plus a `readonly=` option | containment built; no read-only option |
| guest module | about 250–300 lines | 1,724 lines of C and 507 of assembly |

<!-- doccrate:keep-together:end -->


The module is larger than planned mostly for honest reasons. ROM safety needed hand-built
string routines and a `swix` in assembly. The desktop needed a filer, free space, directory
creation and paging. And every error now says what failed.

## Open, as the project states it

- **Filing system number 220 is unallocated**, and error numbers are ad hoc. Both need
  registering with RISC OS Open before anything ships outside the team.
- **No wildcards** in the calls that write catalogue information, and untyped files lose
  their load and execute addresses.
- **Time zones**: UTC on the wire, and the zone in the guest, is the stated fix, and the code
  still adds the host offset (chapter 5).
- **`*Free` from a HostFS directory** is FileCore's command, so it does not work there.
- **Not built**: a read-only option, several shares selected by disc name, zero-copy
  transfers.
- **The shared card images** still load HostFS 1.01 unconditionally over the ROM's copy.
- **The filer uses the hard-disc icon.**

## Open, found by reading the code

Every item here is **INFERRED**: read from the code at `e7e6deffae`, and not tested.

### A create through a symlink can leave the share

`host_path` resolves a guest path component by component under the share root. It then
checks containment with `realpath`, so that a symlink pointing out of the share is
refused. The check only applies when `realpath` succeeds:

```c
char *real = realpath(cur, NULL);
char *rootreal = realpath(s->root, NULL);
bool ok = true;

if (real && rootreal) {
    size_t rl = strlen(rootreal);
    ok = strncmp(real, rootreal, rl) == 0 &&
         (real[rl] == '\0' || real[rl] == G_DIR_SEPARATOR);
}
```

`realpath` fails for a path that does not exist yet, so `ok` stays true. Suppose the
share contains a symlinked directory pointing outside it. Then a **create or rename to a
new name inside that directory** passes the check, and the host follows the link. Reading
an existing file through the same link is refused correctly.

The guest cannot create symlinks itself, so this needs a link already present in the
share. The design's own security section asks for symlinks to be resolved before the
containment check. A fix would resolve the *parent* directory when the leaf does not
exist yet.

The build of 12 September had a second route out. A path component of `//` became `..`
on the host. Version 2.02's escaping of trailing dots, added for Windows, turns that into
an ordinary name, and closes the route incidentally.


<!-- doccrate:keep-together:start -->

#### Found by reading the code, continued

| Where | What |
|:---|:---|
| guest memory access | debug translation checks no page protection, so file data can be written into pages the client itself could not write, including the ROM's copy in RAM |
| snapshots | the device holds host file descriptors and has no migration state; a `loadvm` into a new process with files open would find them gone |
| directory listings | a host name of 40 bytes or more is silently left out |

<!-- doccrate:keep-together:end -->


<!-- doccrate:keep-together:start -->

#### Found by reading the code, concluded

| Where | What |
|:---|:---|
| names | `notes` and `notes,ffb` in one directory both list as `notes` |
| the version 1 header | the error-number and PSR words are defined, and never written |
| the trace | has no off switch |

<!-- doccrate:keep-together:end -->


## What could be shipped

The boot design ends with the question every emulator of RISC OS eventually meets: what
could be given to someone else? It frames its answer as a summary of the licences, and
says plainly that it is not legal advice:

- **The ROM** is Apache 2.0 for most of RISC OS, but not all of it. Shipping one needs an
  inventory of what is inside.
- **RISC OS Direct images must not be redistributed.** They carry commercial third-party
  applications.
- **Broadcom's firmware** carries a use restriction, but QEMU never runs it, so an
  emulator image can simply leave it out.
- **The name** "RISC OS" is a trademark. Describing software as running RISC OS is
  ordinary use; naming a product after it needs permission.
- **The recommendation**: build the boot tree from the Apache-licensed sources, and ask
  RISC OS Open and RISC OS Developments directly. As the design puts it, a written yes is
  worth more than any reading of a licence page.

One provenance note belongs to this document as well. The public design notes behind
HostFS quote the RISC OS programmer's reference and RISC OS Open's filing-system guide
closely, sentence by sentence, because that is how their claims stay checkable. Some
module source comments do too. This document paraphrases every such passage, and
quotes only the project's own words.
