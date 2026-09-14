# 9. Measured, and open

This last chapter gathers the numbers from the whole document in one place, each
with its source. It then lists what is still open: first as the project states it,
then as found by reading the code. It ends with the places where the fork's own
documents have fallen behind its code, so a reader knows which to trust.

## The numbers


<!-- doccrate:keep-together:start -->

### Display

| Measure | Result | Source |
|:---|:---|:---|
| Windows decode against the guest framebuffer | 84 of 480,000 pixels differ: the clock | `418847ac82` |
| Intel Mac decode against a QMP screendump | 134 of 480,000 differ: the pointer, mid-move | `cf4a309fcd` |
| cost of the window to the guest | about 1,580 → 1,540 M/s, −2.3%, within noise | sprint U4 record |
| rows changing between frames at rest | 0.1% | `6c26b418f0` |
| Mac frames copied with the settle rule | 25.0% of 4,500 | `6c26b418f0` |
| Windows, gated against ungated, 1920 × 1080 idle | 1,309 ms against 2,141 ms of copying | `b3d60dbe6a` |

<!-- doccrate:keep-together:end -->


<!-- doccrate:keep-together:start -->

### Pointer and scripting

| Measure | Result | Source |
|:---|:---|:---|
| pointer traffic in one boot from the card | 69 shape updates, about 356 move transactions | graphics design §8a |
| opaque texels in the arrow | 134 of 32 × 32 | graphics design §8a |
| Apple Event round trip against QMP | 16.7 ms against 0.1 ms | scripting design, Q2 |
| `screendump` against `screenshot` | 40 ms against 72 ms | scripting design |

<!-- doccrate:keep-together:end -->


<!-- doccrate:keep-together:start -->

### The blitter

| Measure | Result | Source |
|:---|:---|:---|
| sprite plot, guest against host | 490 µs against 80 µs: **6.1×** | `a3f6b6fafe` |
| sprite pixels on the host | **94.3%** | `8b1d43fb4b` |
| correctness | 0 of 2,304,000 pixels differ | `a3f6b6fafe` |

<!-- doccrate:keep-together:end -->


<!-- doccrate:keep-together:start -->

#### The blitter, continued

| Measure | Result | Source |
|:---|:---|:---|
| full-screen 9 MB fill, `memset` against synchronous Metal | 68 µs against 510 µs | `tools/fillbench.m` |
| one contiguous 9 MB write against row by row | 68 µs against 118 µs | `riscos_blitter.c` comment |
| 8 bpp title-bar fill; full-screen erase | 2 µs; 303 µs | `0223e6f2d8` |
| ROM-spliced module, one boot | 26 fills at 1–2 µs each | `c0a55e4f44` |

<!-- doccrate:keep-together:end -->


<!-- doccrate:keep-together:start -->

### Sound

| Measure | Result | Source |
|:---|:---|:---|
| delivery, the sound card as clock | 29.95 s of 30.0 s: **0.998×** | sound design §12 |
| delivery, virtual clock (superseded) | 0.76× and 0.71× | sound design §10 |
| buffers before the slot fix | 1,265, then a stall | sound design §10 |
| buffers after the slot fix | 5,883 in 100 s | sound design §10 |
| WAV capture of a 91 s boot | peak 16,302 of 32,767 | `e61d3314bd` |

<!-- doccrate:keep-together:end -->


## Open, as the project states it

- **GVFill declines masked sprites** and any plot with plot-action bit 4 set. The
  meaning of bit 4 is not established.
- **Packed sprite sources** through a colour table exist on the host, and nothing
  reaches them.
- **Cross-depth plots**, which include every desktop sprite onto an 8 bpp screen, are
  passed to SpriteExtend by design.
- **The Mac's settle-copy saving** is capped at 75%, because text and lines never
  raise the damage flag.
- **Fill in the project's own ROM** remains the long-term design. The asynchronous GPU
  blit is gated on a measurement never taken.
- **In the ROM, GVFill never sees a sprite plot**, so it is loaded from the share
  instead.
- **Sound:** volume is acknowledged but not applied; there is no capture; the ring is
  not migrated.
- **The screen-bank design** is unbuilt.
- **Mac pointer grab, full screen and the snapshot menu** are unexercised.
- **Sprint 13's wall-time targets** — a Pinboard drag, a NetSurf scroll — have no
  recorded result. The redraw benchmark tool exists, but no run of it is recorded.


<!-- doccrate:keep-together:start -->

## Open, found by reading the code

Each item below is **INFERRED**: read from the code, not tested.

| Where | What |
|:---|:---|
| `bcm2835_fb_get_config` | the seqlock reader's `continue` inside a `do … while` jumps to the loop test; if a write is in flight and the generation has not moved, it returns without filling `*out` |
| `bcm2835_vchiq_get_cursor` | staleness is checked before the caller copies the 64 × 64 image, so a commit in between could tear the sprite for a frame |
| `vmstate_bcm2835_vchiq` | lists `disp_res_w` and `disp_res_h` twice |
| the damage word | DMA copies and kernel text and lines never raise it, so that drawing updates at the 4-frame timeout: 15 Hz at 60 Hz |

<!-- doccrate:keep-together:end -->


<!-- doccrate:keep-together:start -->

#### Found by reading the code, continued

| Where | What |
|:---|:---|
| the Metal front end | gates on the damage flag even when GVFill is not loaded; the Windows twin guards against that |
| `riscos_blitter.c` | the host copy and the mapped physical sprite source are never reached by shipped guest code |
| `bcm2835_dma.c` | a comment says RISC OS uses DMA pattern fills for FillRectangle, while the graphics design's reading of the sources says nothing uses that mode |
| the blitter on an Intel Mac | shared Metal buffers imply a bus transfer on a discrete GPU; the frame cost there is unmeasured |

<!-- doccrate:keep-together:end -->


<!-- doccrate:keep-together:start -->

## Where the documents have fallen behind

The fork's documents are unusually thorough, but they were written sprint by sprint.
Where a document and the code disagree, the code and the later commit messages are
right:

| Document | Says | The code, since |
|:---|:---|:---|
| both blitter READMEs | 98.2% of sprite pixels | 94.3%, since the masked-sprite backout `8b1d43fb4b` |
| emulator README | the Windows front end paces to vsync and fingerprints rows | both removed, `c5f7bb86b7` and `b3d60dbe6a` |
| macOS record | Direct3D reads a texture; middle-click grabs | a byte buffer since `ec4ace18b4`; all buttons are the guest's since `295ea58e4d` |

<!-- doccrate:keep-together:end -->


<!-- doccrate:keep-together:start -->

#### Where the documents have fallen behind, continued

| Document | Says | The code, since |
|:---|:---|:---|
| `ui/metal.m` comment | middle click captures the pointer | as above |
| UI design | raw-input relative mouse | an absolute tablet since `a59d153d17` |

<!-- doccrate:keep-together:end -->


<!-- doccrate:keep-together:start -->

#### Where the documents have fallen behind, concluded

| Document | Says | The code, since |
|:---|:---|:---|
| `riscos_blitter.h`, `.c` | sprite rows are stored bottom first | the path in use runs top down, `52550e8d9a` |
| `riscos_blitter.c` fill comment | the Metal front end copies every frame | the settle rule, `6c26b418f0` |
| GVFill header comment | the RMA is identity mapped; storage lives in the module body | neither: the device header says otherwise, and 1.03 uses an RMA workspace |
| `qapi/ui.json` | vsync pulses follow a presented frame | the generator is independent of rendering |
| the design record's render table | fill parameters as left, bottom, right, top | GVFill reads left, top, right, bottom, and draws correctly |

<!-- doccrate:keep-together:end -->


The public-facing site repository also still says Windows has no sound, and gives the
old middle-click grab and a `Shift+F10` Menu key that the Windows front end does not
handle.

## A note on provenance

The design notes behind this document quote their sources closely. That is how they
stay checkable. It also means several public notes carry passages of RISC OS Open's
own text: BCMSound's description of itself and parts of its assembly in the sound
design, a kernel comment in the screen-bank design, and a video-driver comment in
the UI design. The emulator README states that no RISC OS Open material is included,
and the graphics design states that nothing of the sources is copied into the tree.
Those passages contradict both statements. This document paraphrases every such
passage, and quotes only the project's own words.
