# 5. Moving drawing to the host

Under emulation the slowest part of the machine is the CPU, and RISC OS does most
of its drawing on the CPU. A large window repainting in visible pieces is the
symptom. This chapter covers the host side of the fix: why it is not the host GPU,
the device the guest talks to, and how that device performs fills, copies and sprite
plots.

<!-- doccrate:keep-together:start -->

## What RISC OS offers a video driver

RISC OS 5 gives its video driver a way to take over drawing: *GraphicsV reason 13,
Render*. The kernel tries it before plotting in software. There are exactly three
operations:

| Operation | Issued for | The Pi's driver |
|:---|:---|:---|
| NOP | before software plotting, to let a pending operation finish | waits for its DMA channel to go idle |
| CopyRectangle | window drags, scrolls, block copies, text scrolling | **does it**, with the DMA controller in 2D mode |
| FillRectangle | window backgrounds and borders, clear screen | **declines**; the kernel fills each row in ARM |

<!-- doccrate:keep-together:end -->

Sprites take a different path. The kernel hands plots to the *SpriteV* vector, where
SpriteExtend generates a bespoke ARM routine for each sprite format and caches only
eight of them. Under emulation all of that is guest CPU work.

<!-- doccrate:keep-together:start -->

## The plan, and what shipped

The graphics design, written early on 11 September, planned a module
called *FastSpr* for sprite plots. It would ring the HostFS doorbell device, and fill
would be deferred. Its argument was that claiming FillRectangle means layering a
filter driver over the ROM's video stack, work better done properly in the project's
own ROM. It also planned an asynchronous Metal path. What shipped twelve hours later
differs on three counts:

| Planned | Shipped |
|:---|:---|
| sprites through the HostFS doorbell | a **separate, register-programmed device**, `riscos-blitter` |
| fill deferred to an own-ROM era | **fill claimed after all**, by an ordinary vector claim that checks the reason and operation |
| an asynchronous path on the host GPU | **no GPU at all** |

<!-- doccrate:keep-together:end -->

<!-- doccrate:keep-together:start -->

## Why not the GPU

On Apple Silicon the GPU shares memory with the CPU. It could fill the guest's
framebuffer in place, with no transfer. So the obvious objection had to be
*measured*, and `tools/fillbench.m` did it on an M4:

| Rectangle | memset, per row | memset, one call | Metal, synchronous | Metal, pipelined |
|:---|---:|---:|---:|---:|
| full screen, 9 MB | 118 µs | 68 µs | 510 µs | 132 µs |
| a window, 1.9 MB | 49 µs | — | 232 µs | 27 µs |
| an icon row, 6 KB | 0.1 µs | — | 149 µs | 20 µs |

<!-- doccrate:keep-together:end -->

The benchmark's conclusion: **the cost is latency, not bandwidth.** A GPU dispatch
costs about 150 µs to submit and wait for. That cannot be pipelined away, because
the blit has to be synchronous: the guest plots text and sprites into the same
framebuffer on the very next instruction. Against a real redraw — thirteen small
rectangles and two full-screen clears — that is about 81 µs of `memset` against
about 2.6 ms of Metal.

The benchmark still left a mark on the device. It showed that back-to-back rows want
one big write rather than one per row, and `blit_contiguous()` came from that.

## The device: `riscos-blitter`

The device is new, invented for this purpose, and models no real BCM2711 hardware.
It sits at `0xFD404000`, one page past the HostFS doorbell, in the low peripheral
window. The guest maps its page with `OS_Memory 13` instead of assuming an address.
RISC OS builds its logical memory map from what the HAL asks for, so an unclaimed
page simply aborts.

<!-- doccrate:keep-together:start -->

### The registers

| Offset | Register | Meaning |
|:---|:---|:---|
| `0x00` | `MAGIC` | reads `'BLIT'` |
| `0x08` | `FEATURES` | fill, copy, sprite, mask, table |
| `0x0c` | `GO` | write: run; read: the last status |
| `0x10`–`0x14` | `OP`, `FLAGS` | NOP, FILL, COPY or SPRITE; addressing flags |
| `0x18`–`0x1c` | `DEST`, `SRC` | framebuffer byte offset; source address |
| `0x20`–`0x2c` | `WIDTH`, `HEIGHT`, `DSTRIDE`, `SSTRIDE` | geometry; signed row strides |

<!-- doccrate:keep-together:end -->

<!-- doccrate:keep-together:start -->

#### The registers, continued

| Offset | Register | Meaning |
|:---|:---|:---|
| `0x30`–`0x40` | `PATTERN` (4 words), `PATLEN` | a fill pattern, repeating every 1 to 16 bytes |
| `0x44`–`0x58` | `DSTX`, `DSTY`, `CLIPX0`…`CLIPY1` | sprite position and clip rectangle, top-left origin |
| `0x5c`–`0x6c` | `BPP`, `MASK`, `MSTRIDE`, `SRCBPP`, `TABLE` | sprite depth, mask and colour table |

<!-- doccrate:keep-together:end -->

The status codes are `OK`, `BADOP`, `BADGEOM`, `NOFB` (no framebuffer),
`RANGE` (a row fell outside the framebuffer) and `FAULT` (a source row would not
translate).

### Three design decisions, from the header

**Registers, not a descriptor block.** A block in guest RAM would be one MMIO write
instead of eleven. But it would have to be named by *physical* address, and a RISC OS
module holds a *logical* one. The module area is not identity mapped, so the
translation costs an `OS_Memory` call per rectangle, with cache maintenance. The
header's verdict: eleven writes cost about a microsecond, which is nothing beside the
fill they replace.

**Offsets, not addresses.** With the `BLIT_F_FB` flag, the guest gives byte offsets
*into the framebuffer*, and the host adds the base it already knows. That is the only
addressing RISC OS needs here, and it saves the guest a translation. It also lets
every row be checked against the framebuffer's extent, so a wrong rectangle **spoils
the screen rather than memory belonging to something else**.

**Sprite sources are guest-virtual.** A sprite lives in a sprite area. RISC OS
dynamic areas are virtually contiguous but physically scattered, so there is no one
physical address to hand over. With `BLIT_F_SRC_VIRT`, the host walks the guest's
page tables itself. That costs the guest nothing and handles the scatter for free.

### Synchronous, with the guest stopped

A blit runs inside the MMIO write to `GO`, on the vCPU thread, with the guest
stopped. The device therefore refuses geometry no screen could need, because a wild
height would hang the guest rather than fault it:

```c
#define BLIT_MAX_WIDTH    (1u << 16)
#define BLIT_MAX_HEIGHT   (1u << 14)
#define BLIT_MAX_BYTES    (64u << 20)
```

## Fill

The fill checks the rectangle against the framebuffer, then takes the fastest of
three routes. The common case is a pattern of one repeated byte: the white and grey
window backgrounds. For that case a `memset` beats building a row:

```c
solid = true;
for (uint32_t i = 1; i < s->patlen; i++) {
    if (pat[i] != pat[0]) {
        solid = false;
        break;
    }
}

blit_span(s, s->dstride, &lo, &span);
host = blit_map(dest + lo, span, &mapped);
if (host) {
    uint8_t *rowpat = NULL;         /* built on first unsolid row */

    for (uint32_t y = 0; y < s->height; y++) {
        uint8_t *to = host + ((int64_t)y * s->dstride - lo);

        if (solid) {
            memset(to, pat[0], s->width);
            continue;
        }
        ...
        memcpy(to, rowpat, s->width);
    }
    blit_unmap(host, mapped);
} else if (blit_contiguous(s, s->dstride)) {
    ...                             /* 1 MB chunks through the address space */
} else {
    ...                             /* a row at a time */
}
```

In order of preference:

1. **Map the whole strided span once** with `address_space_map`, and write rows with
   plain stores.
2. **If it will not map and the rows run back to back**, write in 1 MB chunks. That
   applies only when the pattern length divides the width, so a continuous run and
   a row-by-row fill put the same byte in the same place.
3. **Otherwise**, write one row at a time through QEMU's DMA write.

Unmapping a span also marks the gaps between rows as dirty, about twice the bytes
actually written. A row-by-row variant that dirtied less was tried. A narrow fill
then cost 1.00 µs instead of 0.52 µs, and neither front end reads the dirty bitmap,
so it was reverted.

The range check needs only the first and last rows. The stride is constant, so rows
run monotonically through memory. An earlier version walked every row: a thousand
iterations to validate a full-screen fill.

## Copy

A copy maps both spans. **Disjoint ends**, such as a copy between two screen banks,
copy directly: one `memcpy` when contiguous. **Overlapping ends** — a window moved
within the same framebuffer — bounce each row through scratch memory, or use one
`memmove` when contiguous. The DMA paths remain as the fallback.

**INFERRED, from the guest code:** GVFill never issues a COPY. Window copies still
go through the video driver's DMA path (chapter 6 of the
[QEMU A72 walkthrough](../QemuA72Walkthrough/06-time.md)). So the
host copy is ready for the screen-banks plan in chapter 7, but no shipped guest code
reaches it yet.

## Sprite

A sprite plot is rows of pixels from guest RAM into the framebuffer. The host clips
the sprite against the caller's clip rectangle and the screen, all inclusive, with a
top-left origin. The header explains the choice: a rectangle intersection is easy to
get right in C and fiddly in relocation-free ARM.

The expensive part turned out to be reading the source. Translating a row at a time,
as a debug interface invites, is far too slow: a page-table walk per row. Mapping a
page at a time was still slow: a megabyte of sprite is 256 maps. So the source
reader **maps as much as the address space will give in one call**, then walks
forward, extending the run while the guest's pages stay physically adjacent:

```c
if (w->valid && va >= w->va_base + w->run_len) {
    /* Just past the run: try to grow it rather than remap. */
    uint64_t next = w->va_base + w->run_len;
    hwaddr pa;
    MemTxAttrs attrs;

    while (w->run_len + 0x1000 <= w->maplen &&
           va >= w->va_base + w->run_len &&
           blit_src_xlate(w, next, &pa, &attrs) &&
           pa == w->pa_base + w->run_len) {
        w->run_len += 0x1000;
        next += 0x1000;
    }
}
```

`blit_src_xlate` is a wrapper around QEMU's `cpu_translate_for_debug`, which walks
the guest's page tables without faulting. As the comment beside the walker puts
it, translation is the cheap half and mapping is the half worth avoiding.

The destination span is mapped only when the rows nearly touch, when `w × bpp × 2 ≥
pitch`. Otherwise rows are written one at a time, which dirties exactly what
changed. Paths for masked sprites and for packed sources through a colour table
exist on the host, but chapter 6 explains why the guest does not use them.

## GO, and the damage word

Everything funnels through one function, run by the write to `GO`:

```c
static void blit_go(RISCOSBlitterState *s)
{
    int64_t t0 = g_get_monotonic_time();
    uint32_t rc;

    if (!blit_geom_ok(s)) {
        rc = BLIT_RC_BADGEOM;
    } else if (s->op == BLIT_OP_SPRITE) {
        rc = blit_sprite(s);            /* geometry is in pixels, not bytes */
    } else if (s->width == 0 || s->height == 0) {
        rc = BLIT_RC_OK;                /* nothing to do, but not an error */
    } else {
        ...                             /* FILL, COPY, SPRITE or BADOP */
    }

    if (rc == BLIT_RC_OK && s->width && s->height) {
        riscos_blitter_note_damage();
    }
    trace_riscos_blitter_go(s->op, s->width, s->height,
                            s->dstride, s->sstride, rc,
                            (uint32_t)(g_get_monotonic_time() - t0));
    s->status = rc;
}
```

The trace point times every blit, and produced most of the numbers below. And
`riscos_blitter_note_damage()` sets one atomic word, which the front ends take and
clear with an atomic exchange: chapter 7 is about what they do with it.

<!-- doccrate:keep-together:start -->

## Measured, step by step

The optimisation sequence is recorded commit by commit. It is a good example of
measuring before each change:

| Step | Sprite plot, guest vs host | Commit |
|:---|:---|:---|
| first host plot | SpriteExtend 460 µs, host 110 µs: **4.2×** | `bbb1aefe06` |
| framebuffer device found once, not per call | about 40 µs per call gone | `fce3b29fb1` |
| coalesced source maps; VDU constants cached | 490 µs against 80 µs: **6.1×** | `a3f6b6fafe` |
| inside the 80 µs | 61.8 µs host blit, about 18 µs guest overhead | `a3f6b6fafe` |
| after the device review, on a Mac | about 61 µs against about 490 µs; *wants a rerun* | `11ad1ee7c2` |

<!-- doccrate:keep-together:end -->

**INFERRED:** `*SprBench` times a thousand plots with the guest's centisecond clock,
so it resolves 10 µs per plot. A figure such as "about 61 µs" is finer than that clock
can measure, so treat the Mac figure as approximate until the planned rerun.

<!-- doccrate:keep-together:start -->

#### What the host takes on

The share below is 94.3%, although both READMEs still print 98.2%. That figure came
from the masked-sprite build, which was backed out because it drew the desktop wrong
(chapter 6).

| Measure | Result | Commit |
|:---|:---|:---|
| share of sprite pixels plotted by the host | **94.3%** | `a3f6b6fafe`, `8b1d43fb4b` |
| correctness against the same scene without the module | **0 of 2,304,000 pixels differ** | `a3f6b6fafe` |
| one desktop redraw | 15 fills, 2 of them full-screen | `35eb1d8836` |
| a session's fills | 5,878, mean 108 KB, only 11 whole-row | `c2b826a5f6` |
| 8 bpp window furniture | a 91 × 17 fill in 2 µs; a 1920 × 1200 erase in 303 µs | `0223e6f2d8` |

<!-- doccrate:keep-together:end -->
