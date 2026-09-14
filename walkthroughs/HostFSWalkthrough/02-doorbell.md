# 2. The doorbell

Every HostFS operation crosses from the guest to the host through one small QEMU
device, `vmchannel`. This chapter walks through it: the registers, the request
block, why every command runs synchronously, and why the device ended up in two
places in the memory map. It also covers the module code that rings it, and a
struct bug that left every error blind.

## Nothing asynchronous, on purpose

The first design states the shape in one paragraph. The guest writes a request
block into its own RAM, pokes the doorbell with the block's physical address, and
reads a status register. QEMU executes the command **synchronously, inside the MMIO
write, under the big QEMU lock** — the same way the framebuffer's mailbox tags are
handled.

The design is candid about the trade:

> A slow host disc stalls the vCPU for the duration, which is exactly what a slow
> disc does on real hardware; in exchange there is no ring, no interrupts, no
> reorder window, nothing to migrate beyond four registers, and no way for guest
> and host views of a request to disagree.

<!-- doccrate:keep-together:start -->

## The registers

The device is a 16 KiB region with five registers:

| Offset | Register | Meaning |
|:---|:---|:---|
| `0x00` | `MAGIC` | reads `'VMCH'`, so a module can probe safely |
| `0x04` | `VERSION` | the protocol version, 1 |
| `0x08` | `FEATURES` | file operations (only with a share), console, time, v1 frames, logical addresses |
| `0x0c` | `CMD` | write the request block's physical address: **this write is the doorbell** |
| `0x10` | `STATUS` | always reads 1, because the work is already done |

<!-- doccrate:keep-together:end -->

The write handler is the whole transport:

```c
static void vmchannel_write(void *opaque, hwaddr offset, uint64_t value,
                            unsigned size)
{
    VMChannelState *s = VMCHANNEL(opaque);

    vmch_trace("vmch WR off=%llx sz=%u val=%08llx\n",
            (unsigned long long)offset, size, (unsigned long long)value);

    if (offset == VMCH_CMD && size == 4) {
        vmchannel_do(s, (hwaddr)(uint32_t)value & ~(hwaddr)0xf);
    }
    /* every other register is read-only */
}
```

`FEATURES` sets its file-operations bit only when a share root was configured, with
`-global bcm2838-peripherals.vmchannel-root=DIR`. Without a root the device still
answers, so a module can probe for it. The launch scripts set the root from the
`RISCOS_HOSTFS` environment variable.

<!-- doccrate:keep-together:start -->

## The request block

A request is one page of guest RAM: a 64-byte header, then up to 4,032 bytes of
inline data such as a path.

| Offset | Field | v0 meaning | v1 meaning |
|:---|:---|:---|:---|
| `+0` | `cmd` | the command | a FileSwitch entry, `0x100` and up |
| `+4` | `seq` | a sequence number, for the trace | the same |
| `+8` | `rc` | the result code | the same |
| `+12` | `handle` | a file handle, or open flags | a RISC OS error number |
| `+16` | `arglen` | bytes of inline data | the same |
| `+20`–`+51` | scratch | per-command words | **R0 to R7**, verbatim |

<!-- doccrate:keep-together:end -->

There are two generations of commands in one device. **Version 0** is POSIX-shaped:
`OPEN`, `CLOSE`, `READ`, `WRITE`, `SEEK`, `CAT`, `DELETE` and friends, with
physical addresses. **Version 1** carries a RISC OS register frame. The module
copies R0 to R7 in, rings, and copies them back out, without interpreting them. The
device implements both side by side, so card images carrying the old module keep
working. Chapter 3 explains why version 1 exists.

### One struct, so the two sides cannot drift

The module mirrors the header as one byte-accurate C struct:

```c
/* The request block, mirroring include/hw/misc/vmchannel.h exactly.
 * The device reads BYTE offsets; the previous H_* word indices
 * mismatched every field past CMD, so rc/arglen/handle were read from
 * bytes the device never writes.  One struct, byte-accurate, so the
 * two sides cannot drift again. */
struct vmreq {
    uint32_t cmd;                       /* +0  VMCH_HDR_CMD */
    uint32_t seq;                       /* +4  VMCH_HDR_SEQ */
    uint32_t rc;                        /* +8  VMCH_HDR_RC */
    uint32_t handle;                    /* +12 VMCH_HDR_HANDLE */
    uint32_t arglen;                    /* +16 VMCH_HDR_ARGLEN */
    uint8_t  scratch[44];               /* +20 VMCH_HDR_ARG .. +63 */
    uint8_t  data[VMCH_MAX_ARG];        /* +64 VMCH_HDR_SIZE */
};
```

The comment records a real bug from the first day. The module had described the
header with *word* indices, which matched the device's *byte* offsets only for the
first field. Every later field was read from bytes the device never wrote, so the
result code came from the wrong place, and **every error was blind** (commit
`0a63382fb3`).

## Ringing the doorbell

On the module side, one small function sends every request. It poisons the result
code first, so an unanswered request stays visible instead of a stale success. And the
write to `CMD` returns only after the host has done the work, so reading `rc` on the
very next line is safe:

```c
static uint32_t vmch_go(struct hostws *ws)
{
    uint32_t cmd = req->cmd, r0 = REQW(0);

    req->seq = ++ws->w_seq;
    req->rc = 0xFFFFFFFFu;             /* poison: an unanswered request stays visible */
    vmch_reg(VMCH_CMD / 4) = req_phys; /* the device DMAs physical, not logical */
    if (req->rc != RC_OK) {
        ...                            /* remember the refused request, for the error */
    }
    return req->rc;
}
```


<!-- doccrate:keep-together:start -->

## Proving it without RISC OS

As with the mailbox fix in the QEMU walkthrough, the first test used no RISC OS
at all. `tools/vmchtest.s` is a 196-byte bare-metal program. It rings the doorbell
with a ping and prints what came back over the serial port:

| Printed | What it proves |
|:---|:---|
| `48434d56` | the magic: the guest found the doorbell |
| `00000000` | a poisoned result code was cleared by the device |
| `48434d56` | a scratch word written by the device: device-to-guest writes work |
| `11223344` | the argument bytes echoed: both directions of memory access work |
| `00000001` | `STATUS` |

<!-- doccrate:keep-together:end -->

## Where the device lives, and where it had to move

The design needed a physical address that RISC OS's HAL does not name and QEMU does
not map. RISC OS on the Pi reads no device tree, so a hole the HAL does not name
simply does not exist as far as it is concerned. A survey found one:
`0xFD400000`, 20 MiB into the BCM2711's low peripheral window. Nothing in the HAL's
device table lives there.

The bare-metal test reached it. **RISC OS did not.** The module's accesses never
arrived at the device. The reason is how RISC OS builds its memory map: it maps
peripherals on demand, with `OS_Memory 13`, only for the pages the HAL asks for.
An address nothing asked for has no logical mapping.

Commit `0a63382fb3` found a place the guest *does* reach. The system timer at
`0xFE003000` sits in a 1 MiB section that RISC OS has provably mapped as device
memory. So the doorbell is mirrored at `0xFE005000`, an unused hole in that same
section:

```c
/* RISC OS's logical map is built on demand and the FE00 section
 * (system timer at 0xFE003000) is provably live device mapping.
 * The doorbell therefore also lives at 0xFE005000: an unused hole
 * in that same section, so the guest's translations there are real
 * device mappings.  (An alias at 0xFEC00000 was bypassed by the
 * guest: reads there came back right without reaching the device.)
 */
memory_region_init_alias(&s->vmchannel_mr_alias, OBJECT(s),
                         "vmchannel-high",
                         sysbus_mmio_get_region(
                              SYS_BUS_DEVICE(&s->vmchannel), 0),
                         0, 0x1000);   /* one page: a 0x4000 alias
                                       * here would shadow the
                                       * hardware at 0x7E007200 */
memory_region_add_subregion(&s_base->peri_mr, 0x5000,
                            &s->vmchannel_mr_alias);
```

The alias is **one page**, not the full 16 KiB. A larger alias would have shadowed
real hardware two pages further on (commit `00a61e5e7a`). The later blitter device
went the other way: it sits in the low window beside the original doorbell, and its
module maps that exact page with `OS_Memory 13`.

## Mapping on first use

The module maps the doorbell page itself, and it does so **lazily**, on the first
filing-system call. `OS_Memory 13` fails when called from a module's initialisation,
because the IO area is off limits while a module is loading. The same call works
from any ordinary context. The first version reported *OS_Memory 13 failed* at load
time until this was understood:

```c
static _kernel_oserror *ensure_vmch(struct hostws *ws)
{
    ...
    if (vmch) {
        return NULL;
    }
    r.r[0] = 13;                     /* OS_Memory: MapIOPermanent */
    r.r[1] = (int)VMCH_PHYS;
    r.r[2] = (int)VMCH_SIZE;
    e = swix(XOS_Memory, &r, &r);
    if (e != NULL || r.r[3] == 0) {
        return &err_nomap;
    }
    v = (volatile uint32_t *)r.r[3];
    if (v[VMCH_MAGIC / 4] != VMCH_MAGIC_VALUE) {
        return &err_badmagic;
    }
    if (!(v[VMCH_FEATURES / 4] & VMCH_FEATURE_FS)) {
        return &err_nofs;
    }
    if ((v[VMCH_FEATURES / 4] & (VMCH_FEATURE_FSENTRY | VMCH_FEATURE_VIRTADDR))
        != (VMCH_FEATURE_FSENTRY | VMCH_FEATURE_VIRTADDR)) {
        return &err_oldhost;
    }
    ...                              /* translate the request page once, OS_Memory 0 */
    vmch = v;
    return NULL;
}
```

The order of the checks is deliberate:

1. **The pointer is published only after magic and features validate.** A mapping
   onto the wrong page can never be cached.
2. **Each failure has its own error**: no mapping, bad magic, no file support, or
   an old host.
3. **Version 1 is required, not preferred.** The version 0 transfer path translated
   addresses in the guest, and fell back to the identity mapping when that failed,
   writing file data wherever that landed. The source comment's verdict: falling
   back to it silently would be worse than not loading.
4. **The request block is one page-aligned page**, so a single logical-to-physical
   translation covers every byte the device reads.

This laziness turned out to matter again later. When HostFS moved into the ROM, its
initialisation runs even earlier in the boot, and mapping on first use is what
lets it work at all (chapter 6).

## Every access, traced

The device writes every MMIO access and every command to a trace file, flushed line
by line. Trace lines carry the command, result, path and, since commit
`509ab9d478`, the host time the request took. That trace is how the boot was
measured in chapter 8.

**INFERRED:** the trace has no off switch. One recorded instance's trace file is 31
MB. That is fine for a development tool and worth a flag before it ships.
