# 7. Booting from the host

The last substitution goes furthest: no disc image at all. RISC OS boots, reads
its `!Boot` sequence and reaches the desktop from a directory on the host. This
chapter covers it as the principle's end point; the full design is in the
[HostFS walkthrough](../HostFSWalkthrough/index.md).

## The chicken-and-egg problem

A filing system is a module. RISC OS looks for `!Boot` very early. So the filing
system that holds `!Boot` must already be running when that search happens — and
it cannot be loaded *from* a filing system that does not exist yet. The boot
design record calls the soft-load route exactly what it is:

> *soft-loading it from a filing system is a trap*

## Splicing into a stock ROM, without rebuilding it

Chapter 1's first commitment was not to rebuild the ROM. The answer here keeps
that commitment in spirit while relaxing its letter.

A tool, `mkrom.py`, **adds modules to the stock ROM's module chain without
growing the image or touching its header.** New modules overwrite the chain's
terminator and fit into slack space the image already contains — 268 KiB of it in
the 5.31 ROM — while the footers stay byte-identical. The tool refuses if the
slack runs out. The ROM is not recompiled; it simply gains a module at the end of
the list it already walks at start-up.

The design record had originally proposed the opposite — growing the image and
updating its size field — and assumed about 64 KiB of headroom. That version was
never built; fitting into existing slack avoided moving the CMOS blob and touching
the header at all. The
result is a stock RISC OS image that happens to contain a HostFS module, so the
filing system exists before `!Boot` is sought.

The decision about what goes into the ROM is deliberately minimal:

> *Only the filing system goes in the ROM*

Everything else — including GVFill — loads from the host share afterwards. That
keeps the spliced image small and the set of things that must be ROM-safe as
short as possible, which matters, because ROM-safety turned out to be hard (see
[chapter 8](08-the-bill.md)).

## CMOS on the share, too

Chapter 2's CMOS blob placed by QEMU's loader had one limitation: changes did
not persist. Rather than add an NVRAM device, HostFS now claims the operating
system's byte vector and **saves CMOS to the share after each write**. The
persistence problem was solved by the filing system that was already there,
without modelling any storage hardware.

## The honest reading: what it actually bought

This is the part of the project's records most worth reading, because it
contradicts an earlier prediction and says so.

The boot design had predicted that replacing emulated SD access with the doorbell
would make booting *"one to two orders of magnitude"* faster. The crossings did
fall by orders of magnitude. The boot did not:


<!-- doccrate:keep-together:start -->

| | boot to desktop | host crossings |
|:---|---:|---:|
| from the emulated SD card | 15.7 s | 2,434,544 SDHCI accesses |
| **from HostFS** | **14.4 s** | **2,657 doorbells** |

<!-- doccrate:keep-together:end -->


**Nine hundred times fewer crossings, and an 8% faster boot.**

The reason is the insight chapter 1 set up. Under TCG, an MMIO access to an
emulated device is not an exit to a hypervisor — it is an ordinary function call
inside the emulator. The file-system record:

> *under TCG an MMIO access is a function call, not an exit … under TCG it is a
> footnote*

The 2.4 million SD accesses were cheap, because each was just a call. The boot is
dominated by emulating the CPU running RISC OS, and no amount of I/O efficiency
changes that. The 2,657 doorbells cost 0.103 s of host time — 39 µs each.

The record draws the right conclusion and does not overreach: under TCG the
doorbell is a footnote, but under hardware virtualisation, where every device
access *is* an expensive exit to the hypervisor, the same design becomes the
argument. The boot design record was updated to correct its wall-clock
prediction.

And the smaller costs are measured just as carefully. Scanning for typed file
names on the host produced 722 misses, costing 33 ms of a 14.8-second boot — a
number recorded precisely so nobody spends a week optimising it.

## Why this matters for the principle

It would have been easy to present HostFS boot as a speed win and stop. The
records instead establish the *true* case for it:

- it removes the disc image, so the host edits the guest's files directly
- it makes the boot path a host directory under version control
- it collapses millions of crossings into thousands, which will matter the moment
  the CPU is no longer emulated

That last point is the one worth carrying away. The fake-it-in-software approach
optimises the right thing for a future accelerated host even where it barely
changes today's emulated one.


<!-- doccrate:keep-together:start -->

```mermaid
flowchart LR
%% @id fake-boot
%% @name Booting from a host directory
%% @node rom shape=cylinder stroke=#403364 stroke_width=2
%% @node spl shape=rounded stroke=#14375A stroke_width=2
%% @node hfs shape=hexagon stroke=#0A544E stroke_width=2
%% @node boot shape=rounded stroke=#0A544E stroke_width=2
%% @node desk shape=stadium stroke=#2C440D stroke_width=2
    rom["stock ROM"] --> spl["mkrom.py appends HostFS<br/>to the module chain"]
    spl --> hfs["HostFS exists<br/>before !Boot is sought"]
    hfs --> boot["!Boot, GVFill, CMOS<br/>all from the host share"]
    boot --> desk["desktop:<br/>14.4 s, 2,657 doorbells"]
```

<!-- doccrate:keep-together:end -->


