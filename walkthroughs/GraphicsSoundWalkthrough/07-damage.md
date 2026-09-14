# 7. Knowing what the guest drew

A front end that copies the guest's screen on its own clock will sometimes copy it
in the middle of a redraw. The result is a *torn* frame: part of a window at its old
position and part at its new one, visible as a flash during drags. Four approaches
were tried to decide when a copy is worth taking. This chapter follows them in order,
ends with the design for the fix that would remove tearing altogether, and marks
where that design is still only a plan.

## Attempt 1: copy every frame, and accept the tear

The first Windows front end copied the whole framebuffer on every presented frame,
unsynchronised. Its design record gives the reason: the guest writes its screen under
no lock, and **a torn frame is exactly what a real monitor shows mid-update**. Dirty
tracking was deferred until it was shown to matter.

The cost was measured before anything else was done. With the window's full
pipeline live at 60 Hz, a guest benchmark ran about 2.3% slower, within the noise.
The copy runs on the UI thread and the GPU, off the vCPU's core, so it does not cost
the guest. What remained was purely a question of what the user saw.

## Attempt 2: copy in step with the guest's frame

The next idea, commit `6a48d5b5a4`, came from the video driver's own behaviour. It
holds pending screen updates and flushes them at the half-frame pulse. So the vsync
generator was taught to report its phase, and the front end copied only in the first
half of each guest frame, after one flush and before the next. Uploads fell to exactly
150 per 300 presented frames: a 60 Hz window against a 30 Hz guest.

It was backed out twenty minutes later, in commit `c5f7bb86b7`, and the reason is
worth keeping:

- **The premise was wrong.** The Wimp does not paint on vsync. RISC OS multitasks
  cooperatively, so a task repaints when it next polls, driven by the centisecond
  tick and by mouse events. Only the pointer, an explicit vsync wait and screen-bank
  flips are tied to vsync. There is no quiet phase in the frame to aim a read at.
- **The effect was worse than nothing.** Sampling once per guest frame held each
  captured frame on screen for two host frames, so every torn capture stayed up
  twice as long.

The phase function was kept, because it correctly describes the generator and costs
nothing when unused. A future screen-bank flipper will want it.

## Attempt 3: fingerprint the screen

The Windows front end then went back to reading every frame, and added a probe. It
fingerprinted a spread of rows either side of each copy. If the fingerprints
differed, the guest had written during the copy. That copy went into the buffer not
on screen and was discarded, and the previous whole frame stayed up. At rest, about
**5 copies in 3,000** were splices.

It worked, but it guessed at something another component already knew exactly.

## Attempt 4: ask the device that drew it

The blitter device already knows when the guest draws, because much of the drawing
now goes through it. Chapter 5 showed the one line that records it:
`riscos_blitter_note_damage()`, an atomic word set after every blit that lands. The
Metal front end adopted it first, in commit `6c26b418f0`, and Windows followed in
commit `b3d60dbe6a`, which also deleted the probe.

### Why not QEMU's dirty bitmap

QEMU already tracks which pages of guest RAM change, and that would be complete. But
reading it needs QEMU's big lock, and the front end runs on its own thread. Three
attempts at it triggered assertions in three different places inside QEMU's locking
and memory-transaction code. One of those came from a view function being reached
both with and without the lock held. An atomic word needs no lock at all.

### The measurement that justified it

Over five thousand frames, **0.1% of rows changed between one frame and the next**.
The front end was moving nine megabytes thirty times a second to show the same
picture. The desktop is idle almost all the time, because the Wimp draws only when a
task asks it to.

## The settle rule

Both front ends apply the same rule: **copy when the guest has drawn and then
stopped**. Seeing the damage flag means the guest is mid-redraw, so the previous
whole frame stays on screen. Seeing it stop means the screen has settled, and that is
the moment to copy. This is the Metal version:

```objc
#define METAL_SETTLE_MAX 4          /* frames to wait for a settle */

static bool fb_upload(const MetalFbView *v)
{
    static unsigned held;
    static bool pending;
    bool drawing = riscos_blitter_take_damage() != 0;
    bool settled;

    if (drawing) {
        pending = true;             /* new content, not shown yet */
    }

    settled = pending && !drawing;
    if (fb.uploaded && !settled && ++held < METAL_SETTLE_MAX) {
        ...                         /* a periodic log line */
        return false;
    }
    ...                             /* count settles and timeouts */
    held = 0;
    pending = false;

    fb.ring = (fb.ring + 1) % METAL_RING;
    memcpy([fb.raw[fb.ring] contents], v->fb, (size_t)v->pitch * v->rows);
    fb.uploaded = true;
    ...
    return true;
}
```

`METAL_SETTLE_MAX` bounds the wait at four frames, for two reasons given in the
source. Continuous drawing, such as a drag or a scroll, never settles and still has
to animate. And text and lines are plotted straight into memory without passing
through the blitter, so nothing raises the flag for them. Without a forced copy,
typing would never appear.

### The Windows twin, with three differences

The Windows version applies the same rule, with three deliberate additions:

1. **It gates only once the blitter has been seen.** A guest running without GVFill
   never raises the flag. Gating on it anyway would drop the whole display to the
   four-frame timeout for a guest drawing perfectly normally. The commit notes the
   same hazard is present on the Mac and worth fixing there.
2. **It re-checks after the copy.** If the flag rose while the bytes were being read,
   the copy is a splice, so it is dropped and the previous frame stands.
3. **An environment switch, `DX11_NO_DAMAGE`,** restores the old behaviour, so both
   can be measured inside one binary.

The second difference is visible in the source:

```cpp
/* Painted while we were reading?  Then the bytes we took splice two
 * guest states; keep the last whole frame and take it again. */
if (riscos_blitter_take_damage()) {
    fb_seen_damage = true;
    pending = true;
    if (fb_damage_gate() && fb.have_good && ++held < FB_SETTLE_MAX) {
        fb_dropped++;
        return;
    }
    held = 0;
}

fb.raw_cur = next;
fb.have_good = true;
fb_uploads++;
```


<!-- doccrate:keep-together:start -->

### What it saved

| Host | Measurement | Result |
|:---|:---|:---|
| Mac | frames copied, 4,500 frames with a window opening, scrolling and typing | **25.0%** |
| Windows | 1920 × 1080 idle, gated | 2,676 copied, 1,223 skipped, 1,309 ms copying |
| Windows | the same, ungated | 4,199 copied, 0 skipped, 2,141 ms copying |

<!-- doccrate:keep-together:end -->


The Windows commit states the honest conclusion. A full 8 MB copy costs 0.49 ms, so
copying every frame at 60 Hz costs about 3% of one core, and saving a third of that
saves about 1%. The change is still right: it is less work, it matches the Mac's
design, and it retired a worse mechanism. But it is not where the guest's time goes.

## The blind spot

**INFERRED, from the code:** only the blitter raises the damage word. Two kinds of
drawing never do:

- **window drags and scrolls**, which go through the video driver's DMA copies
- **text and lines**, which the kernel plots straight into memory

With GVFill loaded, that drawing reaches the screen only on the four-frame timeout.
At 60 Hz that is every fourth presented frame, an effective 15 Hz. It matches the
25% copy rate the Mac measured, and the Metal commit names text and lines explicitly.
The pointer is unaffected, because it is composited every frame.

Covering those paths would close the gap. The DMA model could raise the same word
after a 2D copy. **INFERRED:** that is a small change, but nobody has made it.

## The real fix: screen banks

Every scheme so far has had to guess when the guest has finished drawing, and a wrong
guess shows a torn frame. The fork's screen-bank design, `BANKS.md`, aims at making a
wrong guess harmless instead. The idea came out of a session watching the desktop
tear. The design quotes it in one line:

> create fb, display fb, copy fb -> bb, update bb, swap

RISC OS already has the pieces. The kernel sizes its screen memory for two banks,
and there are calls to choose which bank is displayed and which is drawn into. The
Wimp draws through `OS_Plot` and `OS_SpriteOp` and never touches the screen address
directly. The video driver can pan. A live framebuffer at 1920 × 1200 is allocated
at 1920 × 2400: room for a second bank.

The prize is not speed. **A mistimed swap shows a complete older frame instead of a
half-drawn one** — late rather than torn.

### Why the copy is the part that matters

The Wimp redraws *incrementally*. When something changes, it gives each task a list
of damaged rectangles and repaints only those. Flip a bank under that, and the back
buffer holds the frame from two flips ago: the backdrop, other windows and the icon
bar would all snap back to stale content. Copying front to back first makes the back
buffer correct before anything draws into it.

That copy is also why RISC OS does not double-buffer on real hardware: on a Pi it is
9.2 MB of block memory moves every frame. On the host it is one `BLIT_OP_COPY`, which
the blitter implements and has never been asked to perform.

### Per-bank damage, and why it belongs in the Wimp

The design's sharpest argument is about where the fix finally belongs. With two
banks, **a damaged rectangle is not correct until both banks have been repainted**.
Damage stops being a per-frame list and becomes per-bank: each rectangle has to
survive until every bank has seen it. The damage lists live inside the Wimp, and
nothing outside it can see them. So:

- **A module can prove the idea.** It hooks entry to `Wimp_Poll`, when a task has
  finished its work, then swaps banks, copies the whole screen front to back, and
  points VDU output at the other bank.
- **A Wimp that knows it is double-buffered needs no copy at all.** It redraws the
  union of this frame's damage and last frame's into the back buffer. That turns a
  fixed full-screen cost per frame into a cost proportional to what changed, which on
  an idle desktop is nothing.

The design notes this would be worth having upstream: the desktop tears on real
hardware too, and the copy's cost is the only reason RISC OS does not double-buffer.


<!-- doccrate:keep-together:start -->

### The plan, and its status

| Step | What |
|:---|:---|
| 1 | the front end follows the displayed bank instead of the framebuffer base; a no-op while nothing banks |
| 2 | a separate module, not GVFill, so display sync and blitting do not share a fate |
| 3 | drive it from the `Wimp_Poll` pre-filter: swap, copy with `BLIT_OP_COPY`, retarget VDU output |
| 4 | measure: count swaps and copy cost from the blitter trace, and judge tearing by eye |

<!-- doccrate:keep-together:end -->


**None of this is built.** The design lists its open questions first, and insists the
measurements come before code, because every wrong turn so far came from writing code
first. Two gaps are visible from reading the code (**INFERRED**):

- GVFill's fills use framebuffer-relative offsets, so drawing into a second *write*
  bank would need it to follow the screen start address.
- QEMU's framebuffer model caps the virtual height at 2,560 rows, so two banks taller
  than 1,280 rows would be clamped. That matters for the 4K modes a later sprint
  plans.
