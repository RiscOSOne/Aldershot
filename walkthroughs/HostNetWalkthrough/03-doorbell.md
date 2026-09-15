# 3. A doorbell in the GENET window

Every HostNet request reaches the host through one small QEMU device. This chapter
walks through where the device lives and why, its registers, the 64-byte request
block, the commands and transport codes, the dispatcher that runs when the doorbell
rings, and the switched-off state that makes the window look exactly as it did
before HostNet existed. It ends by comparing the device with HostFS's doorbell.

## A known hole at a known address

The Raspberry Pi 4's Ethernet controller, GENET, sits in the BCM2711's low
peripheral window at guest physical address 0xFD580000. The emulator does not model
it. Until HostNet, that 64 KB window held an *unimplemented device*: a QEMU stand-in
that reads as zero. Its only purpose was that RISC OS's EtherGENET driver, reading
the controller's revision register, would get zero and decline politely instead of
taking an abort. A commit on 10 September, `5a61a7881c`, had moved that stand-in to
the address where RISC OS's hardware layer actually looks for GENET.

With the guest out of networking, EtherGENET is unplugged and the stand-in has
nothing left to do. The header of `include/hw/misc/hostnet.h` describes what that
leaves: "the 64 KB is a known-sized hole at a known address that RISC OS maps the
same way it maps the HostFS doorbell." The same comment is careful to add that
occupying it "is not modelling GENET; it is using the hole."


<!-- doccrate:keep-together:start -->

#### The wiring

This is the wiring, in `hw/arm/bcm2838_peripherals.c`:

```c
    /*
     * The GENET Ethernet MAC is not modelled, and with HostNet it never
     * will be: the guest does no networking below the socket layer, so
     * there is no link for a MAC to drive and EtherGENET is unplugged.
     * What stood here was an unimplemented device present only so that
     * EtherGENET's revision read returned zero rather than taking an
     * external abort.  Its window is now HostNet's doorbell, same
     * address, same size.  Low peripheral window (0xFD580000 to the
     * guest) because RISC OS maps that on request with OS_Memory 13,
     * the way the HostFS doorbell at 0xFD400000 is reached; a page in
     * the FE00 section aborts unless the HAL asked for that exact page.
     */
    object_initialize_child(OBJECT(s), "hostnet", &s->hostnet, TYPE_HOSTNET);
    if (!sysbus_realize(SYS_BUS_DEVICE(&s->hostnet), errp)) {
        return;
    }
    memory_region_add_subregion_overlap(&s->peri_low_mr, BCM2711_GENET_OFFSET,
            sysbus_mmio_get_region(SYS_BUS_DEVICE(&s->hostnet), 0), -1000);
```

<!-- doccrate:keep-together:end -->


The device has no interrupt line. GENET's two interrupt numbers remain defined and
unused. Everything HostNet needs to tell the guest, it tells in the answer to a
doorbell ring, including the wake-ups in chapter 6.


<!-- doccrate:keep-together:start -->

## The registers

The window is 64 KB, and only the first four words mean anything. Every access must be
a 4-byte, little-endian word:

| Offset | Register | Read | Write |
|:---|:---|:---|:---|
| `0x00` | `HN_MAGIC` | `'HNET'`, 0x54454E48 | ignored |
| `0x04` | `HN_VERSION` | 1 | ignored |
| `0x08` | `HN_FEATURES` | bit 0, `HN_FEATURE_SOCKETS` | ignored |
| `0x0C` | `HN_CMD` | 0 | a request block's guest **logical** address: **the doorbell** |
| `0x10` to the end | none | 0 | ignored |

<!-- doccrate:keep-together:end -->


The value written to `HN_CMD` is a logical address, not a physical one. The module
writes the address of its request block as its own code sees it, and the host walks
the guest's page tables to find it. Chapter 4 describes that walk.


<!-- doccrate:keep-together:start -->

## The request block

A request is a 64-byte header in guest memory, followed for one command by a list
of results. The header deliberately mirrors HostFS's version 1 header, "so the two
modules' transports read the same way":

| Offset | Field | Direction | Meaning |
|:---|:---|:---|:---|
| `+0` | `CMD` | guest to host | 0 `PING`, 1 `SWI`, 2 `POLL` |
| `+4` | `SEQ` | guest to host | a sequence number, kept by the host |
| `+8` | `RC` | host to guest | the transport result; the guest pre-fills 0xFFFFFFFF |
| `+12` | `ERRNO` | host to guest | a 4.4BSD errno, 0 for none |

<!-- doccrate:keep-together:end -->


<!-- doccrate:keep-together:start -->

#### The request block, continued

| Offset | Field | Direction | Meaning |
|:---|:---|:---|:---|
| `+16` | `SWI` | guest to host | which socket SWI, 0 to 34 |
| `+20` to `+51` | `R0` to `R7` | guest to host | the SWI's registers, verbatim |
| `+52` | `RESULT` | host to guest | R0 on success; the magic for `PING`; a count for `POLL` |
| `+64` onwards | the poll list | host to guest | up to 16 words, for `POLL` only |

<!-- doccrate:keep-together:end -->


Two details matter. The `RC` field starts as 0xFFFFFFFF, which no answer uses, so
a request the host never answered is visible as such. And socket errors do not use
`RC` at all, as the next table explains.


<!-- doccrate:keep-together:start -->

### Commands and transport codes

| `RC` | Name | Meaning |
|:---|:---|:---|
| 0 | `HN_RC_OK` | the call was made; `ERRNO` says whether it succeeded |
| 1 | `HN_RC_BADCMD` | an unknown command |
| 2 | `HN_RC_BADSWI` | a SWI number of 35 or more |
| 3 | `HN_RC_BADADDR` | a guest address would not translate |
| 4 | `HN_RC_NOSOCKETS` | the device is present but sockets are switched off |
| 5 | `HN_RC_RETRY` | the call would block on a socket the guest believes is blocking: wait and ring again |

<!-- doccrate:keep-together:end -->


The comment on `HN_RC_OK` states the rule: a socket call that fails for an ordinary
networking reason, such as a refused connection, "is not a transport failure, it is
the answer". It comes back as `HN_RC_OK` with `ERRNO` set. The other codes mean the
transport itself could not do what was asked, and `HN_RC_RETRY` is not a failure at
all. Chapter 4 is about that last one.


<!-- doccrate:keep-together:start -->

## The dispatcher

`hostnet_write` passes a write to `HN_CMD` to `hn_ring`, with the block's address.
It runs on the guest CPU's thread, inside the memory write, holding QEMU's global
lock. It starts by reading the header:

```c
static void hn_ring(HostNetState *s, uint64_t base)
{
    uint32_t cmd = hn_ld32(base + HN_HDR_CMD);
    uint32_t regs[8];
    HNReply r = { .result = 0, .err = 0, .rc = HN_RC_OK };
    uint32_t swi;

    s->seq = hn_ld32(base + HN_HDR_SEQ);
    ...
    s->rung = true;
    ...
```

<!-- doccrate:keep-together:end -->


<!-- doccrate:keep-together:start -->

#### The path for a socket SWI

After `PING` and `POLL` have been answered, a socket SWI is checked, made, and
answered:

```c
    swi = hn_ld32(base + HN_HDR_SWI);
    if (swi >= HN_SWI_COUNT) {
        hn_st32(base + HN_HDR_RC, HN_RC_BADSWI);
        return;
    }
    if (!vmch_guest_rw(base + HN_HDR_REGS, regs, sizeof(regs), false)) {
        hn_st32(base + HN_HDR_RC, HN_RC_BADADDR);
        return;
    }
    if (!s->enabled) {
        hn_st32(base + HN_HDR_RC, HN_RC_NOSOCKETS);
        return;
    }

    errno = 0;
    hn_do_swi(s, swi, regs, &r);
    ...
    hn_st32(base + HN_HDR_RESULT, (uint32_t)r.result);
    hn_st32(base + HN_HDR_ERRNO, r.err);
    hn_st32(base + HN_HDR_RC, r.rc);
}
```

<!-- doccrate:keep-together:end -->


The elided parts answer `PING` and `POLL` and reject unknown commands before the SWI
path, and write the trace line after it. `PING` writes the magic number into `RESULT`, which is how
`*HostNetPing` checks that the host is there. `POLL` runs the wake-up scan from
chapter 6.

Every verb fills in an `HNReply` of three fields, the result, the errno and the
transport code, and the dispatcher writes all three back in one place. The `RC`
word is written last. The guest reads it as soon as the memory write that rang the
doorbell returns.


<!-- doccrate:keep-together:start -->

### The `rung` flag

`s->rung` is set on every ring and cleared on reset:

| Where | What it means |
|:---|:---|
| set in `hn_ring` | the module has rung since the machine last started |
| cleared in `hostnet_reset` | a restarted machine has not rung yet |
| read by the Windows window menu | whether the running RISC OS is using HostNet, whatever the module file's folder now says |

<!-- doccrate:keep-together:end -->


The guest rings fifty times a second once the module is loaded, so the flag becomes
true within one tick. Chapter 8 shows the menu using it.


<!-- doccrate:keep-together:start -->

## Switched off: indistinguishable from before

The device's `sockets` property is off unless a launcher passes
`-global hostnet.sockets=on`. The first version answered its magic number even when
off, and only refused socket calls. `50ef1a1bcd` changed that, because the window's
old zero reading had a job to do:

```c
/*
 * Switched off, the whole window reads zero — not just the feature bits.
 *
 * This window used to hold an unimplemented device, and its reading as
 * zero was load-bearing rather than incidental: EtherGENET reads the GENET
 * revision register, which is at offset 0, and takes zero to mean an
 * unknown controller it should decline.  Answering 'HNET' there would give
 * it a revision it has never heard of on a machine that has not asked for
 * HostNet at all.
 ...
 */
static uint64_t hostnet_read(void *opaque, hwaddr offset, unsigned size)
{
    HostNetState *s = HOSTNET(opaque);

    if (!s->enabled) {
        return 0;
    }
    ...
}
```

<!-- doccrate:keep-together:end -->


<!-- doccrate:keep-together:start -->

#### Writes go nowhere

The write handler applies the same rule to the doorbell itself:

```c
static void hostnet_write(void *opaque, hwaddr offset, uint64_t value,
                          unsigned size)
{
    HostNetState *s = HOSTNET(opaque);

    /* Switched off, writes go nowhere: a machine that has not asked for
     * HostNet must not be able to reach it by writing to the window. */
    if (s->enabled && offset == HN_CMD) {
        hn_ring(s, value);
    }
}
```

<!-- doccrate:keep-together:end -->


<!-- doccrate:keep-together:start -->

### The window, read back

The commit records the three identification registers read from the guest in both
states:

| State | `HN_MAGIC` | `HN_VERSION` | `HN_FEATURES` |
|:---|:---|:---|:---|
| sockets off | 0x00000000 | 0x00000000 | 0x00000000 |
| sockets on | 0x54454E48 | 0x00000001 | 0x00000001 |

<!-- doccrate:keep-together:end -->


The stock CMOS unplugs EtherGENET, so nothing was reading the register. The commit
calls that "luck rather than a guarantee": a machine booted with a different CMOS
would have found it. Off now also means unreachable. With sockets off, no guest
program can make the host do anything by writing to this window.

## Reset and snapshots

A machine reset abandons every socket the guest held, so `hostnet_reset` closes them
all and clears the per-socket flags, the sequence number and `rung`. It leaves
`enabled` alone, so a switch turned on in the session stays on.

The saved-state description migrates only the sequence number. Its comment explains
the choice: "A host socket cannot be carried into a saved image, so a -loadvm restore
finds every descriptor gone and answers EBADF. Pretending otherwise would hand the
guest numbers that connect to nothing."

## Two doorbells compared

HostNet copied HostFS's transport, but shares only one function with it. The table
compares the two devices:


<!-- doccrate:keep-together:start -->

#### HostFS and HostNet

| | HostFS, `vmchannel` | HostNet, `hostnet` |
|:---|:---|:---|
| **address and size** | 0xFD400000, 16 KB, plus a one-page alias at 0xFE005000 | 0xFD580000, 64 KB, no alias |
| **magic** | `'VMCH'` | `'HNET'` |
| **header** | 64 bytes; `R0` to `R7` at `+20` | 64 bytes; `R0` to `R7` at `+20`, `RESULT` at `+52` |
| **addresses** | guest logical, walked on the host | the same, through the same function |

<!-- doccrate:keep-together:end -->


<!-- doccrate:keep-together:start -->

#### HostFS and HostNet, continued

| | HostFS, `vmchannel` | HostNet, `hostnet` |
|:---|:---|:---|
| **shared code** | owns `vmch_guest_rw` | calls it |
| **state and sequence numbers** | its own | its own |
| **reset** | flushes and closes open files, since `a74bfcf41d` | closes sockets, since Sprint 1 |

<!-- doccrate:keep-together:end -->


The Windows release adds one more dependency: HostNet's switch is a file on the
HostFS share, so the menu that moves it needs a machine with a share. Chapter 8
covers that.
