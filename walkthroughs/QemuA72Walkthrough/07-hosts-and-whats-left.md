# 7. Two hosts, and what is left

The fork was written on Windows: MSYS2 for the build, Direct3D 11 for the window,
and a Win32 high-resolution waitable timer for the clock. On day two it moved to a
Mac with Apple Silicon, and on day four to an Intel Mac. This chapter covers what
moving it showed about the design, the measurements it made possible, how the work
stands against QEMU's requirements for upstream contributions, and what is still
open.

## The Mac: the emulation was already portable

The macOS port record opens with a one-line verdict:

> **The emulation was already portable, and the front end was not.**

Before any macOS code was written, an unmodified checkout was configured with QEMU's
standard Cocoa display and built with Apple clang. It booted RISC OS 5.30 from the
card to the 800×600 desktop, with NetSurf on the Welcome page, in under 20 seconds
on an M4. The network worked from the first boot. The bare-metal channel-0 test
printed `00000080`. The Python tools read the CMOS file out of the card image and
built the blob unchanged.


<!-- doccrate:keep-together:start -->

#### What was platform-specific

| Piece | Verdict on macOS |
|:---|:---|
| the VCHIQ and channel-0 peers, the vsync generator, the legacy interrupt controller, every device fix | portable; no `#ifdef _WIN32` anywhere in them |
| `riscos-pi4/tools/*.py` | portable |
| the Windows symlink workaround | already gated to Windows; a no-op |
| `system/hrtimer.c` | had a POSIX branch all along; measured in chapter 6 rather than assumed |
| `ui/dx11.{h,c,cpp}` | Windows only, and the whole of the work |

<!-- doccrate:keep-together:end -->


Every RISC OS-specific change is plain C against QEMU's own interfaces. **So the port
was not a port of the emulator. It was a new front end:** `ui/metal.{h,c,m}`, the
Metal twin of the Direct3D 11 window. Both front ends, and the paravirtual blitter
behind them, are covered in the
[graphics and sound walkthrough](../GraphicsSoundWalkthrough/index.md).


<!-- doccrate:keep-together:start -->

## Speed on two hosts

Every macOS figure was measured with a *second* full machine running in a window
on the same Mac, so the Mac numbers are a floor rather than a best case:

| | i7-12700 | M4 | M4 against i7 |
|:---|---:|---:|---:|
| power-on to a screen that stops changing | ~27 s | **20.8 s** | 1.3× |
| centisecond ticker, worst gap | 11.6 ms | 12.5 ms | — |
| `bench`: two instructions, registers only | ~2,000 M/s | 1,673 M/s | 0.84× |
| `bench2`: six instructions, a load and a store | ~500 M/s | 895 M/s | 1.8× |

<!-- doccrate:keep-together:end -->


The split is the interesting part. A two-instruction register loop is almost pure
TCG dispatch, and the i7's clock speed wins it. Add a load and a store, which is
what real code does, and the M4 is nearly twice as fast. The boot involves device
emulation, block I/O and the Wimp rather than a spin loop, and it follows the second
number.

Two more figures place the emulator honestly:

- **The window costs nothing measurable.** Headless, a benchmark ran at about 1,580
  M/s; with the window's full decode-and-present pipeline live at 60 Hz, about 1,540
  M/s. That is −2.3 %, within the ±10 % noise of single samples. The 1.9 MB upload
  per frame runs on the UI thread and the GPU, off the vCPU's core.
- **Against real silicon**, the synthetic figures put a vCPU at *between a quarter
  and a half of one real Pi 4 core*, and worse on NEON-heavy code. That is TCG's
  usual standing, not a cost of this fork.

## An Intel Mac

Commit `cf4a309fcd` took the fork to a Xeon W-3235 running macOS 26. Its problems
were not in the emulator. Homebrew offered no prebuilt packages for several
dependencies on that system, and refused to build them from source while an
outdated Xcode sat at the default path.

The fork's answer was `tools/build-deps-macos.sh`. It builds `pcre2`, `glib`,
`libslirp` and `capstone` statically, with the Command Line Tools alone, outside the
repository, idempotently. Deleting or updating the user's Xcode was deliberately not
made a build requirement.

Verified on that machine:

- the channel-0 regression prints `00000080`
- RISC OS boots from the card to a live, interactive desktop behind `-display metal`
- **the decode is pixel-faithful.** A capture from the Metal surface and a QMP
  screendump taken a second apart differ in **134 of 480,000 pixels**. They are a
  12 × 22 block at dead centre: the pointer, mid-move between the two captures.
  Every other pixel is identical. The Windows acceptance test had the same shape:
  84 pixels, the clock.

One method lesson is recorded so it is not relearned. An earlier comparison showed
89 % of pixels differing. It had compared a periodic capture of one desktop moment
with a screendump of another. The periodic capture only fires on change, so the two
images in a comparison must be taken close together.

## Many cores: measured, then mostly left alone

A question came up on day three: can the emulator notice a host with fast and slow
cores, and keep the busy vCPU on a fast one? It can notice. The measurements said
forcing the choice was the wrong move.


<!-- doccrate:keep-together:start -->

### Windows: pinning loses

Commit `e7699165ab` adds `qemu_thread_prefer_performance_cores()`, called from the
TCG vCPU threads. On an i7-12700, with 8 performance and 4 efficiency cores, each
run was one 30-second RISC OS boot. Retired guest instructions were counted with a
TCG plugin, and the two arms were alternated so thermal drift fell on both equally:

| Arm | Runs, billions of instructions | Mean |
|:---|:---|---:|
| pinned to performance cores | 2.55 / 3.15 / 2.54 | 2.75 |
| left to the scheduler | 3.14 / 3.19 / 3.20 | **3.18** |

<!-- doccrate:keep-together:end -->


**Pinning is about 13 % slower, and much less even.** A hard affinity mask overrides
Intel's Thread Director, which already knows which core is fastest, and takes that
choice away. An earlier single pair had suggested a 37 % *gain*. That was the first
run of the session, on a cold chip, and alternating the arms is what exposed it.

So the default leaves placement to the scheduler, and pinning is available only
through the `QEMU_VCPU_PIN` environment variable. By default the thread only opts
out of Windows' efficiency quality-of-service class. That overrides nothing, and it
measured as neutral (2.94 against 2.96 billion). It is kept for the case that could
not be measured on a desktop: a laptop on battery, where that class is applied far
more readily.

### macOS: a QoS class, not a core

Commit `b46fa9765d` adds the macOS side. Apple Silicon offers no core pinning, but
it has a quality-of-service ladder. A new pthread starts in the default class. It
does **not** inherit its creator's class — that was measured, not assumed. The
helper moves the vCPU threads, the timer thread and QEMU's main loop to
`USER_INTERACTIVE`, which the scheduler keeps on performance cores while still
choosing which one.


<!-- doccrate:keep-together:start -->

#### Quality of service on an M4

Measured with `bench2` on an M4 (4 performance and 6 efficiency cores), in M/s:

| Performance cores also occupied by | Before | After |
|:---|---:|---:|
| nothing | 968–975 | 931–977 |
| 4 default-class busy loops | 870 | 815 |
| 4 interactive-class busy loops | 856 | 803 |
| 12 interactive-class busy loops | 714 | **843** |

<!-- doccrate:keep-together:end -->


It is neutral until the machine is genuinely oversubscribed. Under oversubscription,
one run without the change fell to 451 M/s, which is efficiency-core territory.
With the change, no run fell below 764. That was one occurrence in three runs, so
the record calls it a tendency rather than a rate.

The Xeon has twelve uniform cores, so the helper does nothing there. A darwin
core-affinity arm could matter on a hybrid Intel Mac, but was not built. The
fork's rule is measurement first.

### A fix that fixed nothing

The fork's only change under `target/arm`, commit `1dd95655a9`, came from the
HostFS work. The host was translating guest addresses and reporting some pages
unmapped. That looked like a page-table-walk bug, so domain-fault checks were
skipped for debug translations. The real cause was RISC OS mapping application
space on demand: the pages genuinely were not mapped yet. The commit keeps the
change, and says plainly that it **"demonstrably fixes nothing here, and can be
dropped without loss."**

## Upstreaming: the case for

The commit messages were written to be sent upstream. Most use QEMU's `subsystem:`
subject style, explain the bug in guest-neutral terms, give the trace evidence, and
name who else is affected. The fork identifies nine fixes as plain QEMU bugs.


<!-- doccrate:keep-together:start -->

### Nine candidate fixes

| Fix | Who else it affects |
|:---|:---|
| AArch32 secondary boot stub | any 32-bit guest on `raspi3b` or `raspi4b` |
| I2C byte access | any guest that uses `STRB` on the FIFO |
| I2C start on `ST`, with a TX FIFO | any guest following the documented sequence |
| system timer to the GIC | any guest using the system timer through the GIC on `raspi4b` |
| SD card on EMMC2 | any `raspi4b` guest that uses the card |

<!-- doccrate:keep-together:end -->


<!-- doccrate:keep-together:start -->

#### Nine candidate fixes, continued

| Fix | Who else it affects |
|:---|:---|
| GIC legacy nFIQ inputs | any SoC wiring an older controller into a GIC-400's legacy inputs |
| DMA 2D transfers of any width | any guest using the DMA engine's 2D mode |
| DWC2 `HCHINT` and per-device IRQ level | any driver that defers a channel interrupt by masking it |
| `usb-net rndis=off` | any USB stack that announces a device in its first configuration |

<!-- doccrate:keep-together:end -->


The plan for a give-back sprint adds three more to the series: the SD read-ahead,
the channel-0 peer and the property tags. It keeps the VCHIQ peer, the EDID answer
and the BCM2711 legacy controller in the fork. **INFERRED:** three more look generic
too — the tablet `SET_PROTOCOL` fix, `usb-net` migration, and the timer and
framebuffer `post_load` fixes.

## Upstreaming: what stands in the way

As the code stands, several things would stop a submission. They fall into two
groups. The AI co-author trailers break down as Claude Opus 5 on 82 commits,
Claude Fable 5.1 on 31, ZCode GLM on 5 and Claude Opus 4.8 on 4.


<!-- doccrate:keep-together:start -->

### Process and policy

| Issue | Detail |
|:---|:---|
| **QEMU's policy on AI-generated content** | `docs/devel/code-provenance.rst` declines contributions believed to include AI-generated content; 122 of the 156 commits carry AI co-author trailers |
| **No sign-offs** | no commit carries a `Signed-off-by` line, which QEMU requires under its Developer's Certificate of Origin |
| **No tests** | nothing under `tests/` changed; the channel-0 check is a bare-metal blob outside QEMU's test harness |
| **Not for upstream by its own label** | the Windows symlink workaround says so in its commit message |

<!-- doccrate:keep-together:end -->


<!-- doccrate:keep-together:start -->

### The shape of the code

| Issue | Detail |
|:---|:---|
| **Blast radius** (INFERRED) | the channel-0 peer, VCHIQ, ARM timer, SMI latch and 30 Hz vsync generator are created in the code shared by *every* raspi machine; only RISC OS on `raspi4b` is reported tested |
| **A parallel timer system** (INFERRED, untested) | `system/hrtimer.c` sits outside QEMU's timer lists, so `-icount` and record/replay are likely not honoured |
| **Layering** (INFERRED) | a guest-visible rate, the vsync, is set through a `-display` option; the front ends find the framebuffer by object path |

<!-- doccrate:keep-together:end -->


<!-- doccrate:keep-together:start -->

#### The shape of the code, continued

| Issue | Detail |
|:---|:---|
| **Migration compatibility** (INFERRED) | `bcm2835_i2c` and `bcm2835_property` bumped their version from 1 to 2 instead of adding subsections |
| **Missing licence notices** | `hw/misc/vmchannel.c`, `include/hw/misc/vmchannel.h`, `include/hw/misc/riscos_blitter.h`, `ui/dx11.c`, `ui/dx11.cpp`, `ui/dx11.h` and `ui/metal.h` |
| **Documentation** | `docs/system/arm/raspi.rst` is untouched; the `rndis` and `mode` properties are undocumented |
| **A plan not built** | the design called for a separate `raspi4b32` machine name; it does not exist in the code |

<!-- doccrate:keep-together:end -->


## What is left

### Open limitations

- **`SET_CLOCK_RATE`** is still not implemented; it only logs.
- **The card stall.** About a millisecond per block during boot is measured but not
  understood.
- **GENET and the VL805** USB 3 controller behind PCIe are not modelled. EtherGENET
  still needs the CMOS unplug bit or the patched ROM.
- **The USB start-of-frame FIQ** runs at 1 kHz from QEMU's main loop. It is the next
  candidate for the timer thread.
- **EDID** stops at 1920×1200, with no CTA-861 extension block. The Direct3D 11
  decoder also has a texture-width ceiling of 16,384.
- **Later sprints not started:** `roscc` on the real target, the debugger, benchmarks
  against hardware, record and replay, the upstream series, and packaging.
- **macOS:** pointer grab, full screen and the snapshot menu are unexercised, and
  moving between displays of different DPI is untested. The Apple Events scripting
  interface is built through stage E3; breakpoints, single step and signing, planned
  as E4 to E6, are not built.

### Documents that have fallen behind

- `DESIGN.md` still gives its status as "Sprint 2 done". Its blocker table still
  lists the abandoned EDID EEPROM.
- The fork banner in `README.rst` still says keyboard input is not proven.
- The VCHIQ peer's header comment, and a README passage, still describe a small
  device that refuses every service.
- The README cites a design-record section for the card stall that does not
  contain it.
- The DMA fill comment disagrees with the design record about FillRectangle.

### Risks found by reading, not by testing

- **A seqlock reader that can return nothing.** `bcm2835_fb_get_config` retries
  while the generation is odd. But the `continue` inside its `do … while` jumps to
  the loop condition. If a write is in progress and the generation has not moved
  between the two reads, the loop exits without writing `*out`. The callers in both
  front ends pass an uninitialised stack structure. The window is tiny.
- **Record and replay.** Timer-thread deadlines are likely invisible to `-icount`
  and replay.
- **Other guests.** The new common-code devices could change the behaviour of other
  raspi guests, which nobody has tested.


<!-- doccrate:keep-together:start -->

## The fork at a glance

| Change | Commit |
|:---|:---|
| AArch32 secondary stub, chosen by CPU state | `86a43faa32` |
| mailbox channel-0 power peer | `4f3f926d0c` |
| system timer to the GIC | `57651d54ce` |
| VCHIQ peer; bounds checks | `1013487c14`, `baddb065d3` |
| PCIe and GENET stubs | `1a22c9b0b1`, `418847ac82`, `5a61a7881c` |
| SD card on EMMC2; 64 KiB read-ahead | `a7ea3f48da`, `e48ee2c73a` |

<!-- doccrate:keep-together:end -->


<!-- doccrate:keep-together:start -->

#### The fork at a glance, continued: tags, I2C and interrupts

| Change | Commit |
|:---|:---|
| property tags: buffers, GPIO state | `f87fdd5f72`, `42de159040` |
| EDID: 800×600, then wide modes | `70522977c7`, `7ef62fa51f` |
| I2C: byte access, sub-word access, ST start and TX FIFO | `d742c3b617`, `57724dbd8e`, `59e0d5b5ea` |
| BCM2711 legacy controller; GIC legacy FIQ bypass | `873dccc24c`, `54eac9c9fb` |
| DWC2 `HCHINT` and IRQ level | `622a7bde8c` |

<!-- doccrate:keep-together:end -->


<!-- doccrate:keep-together:start -->

#### The fork at a glance, continued: USB, time, DMA and hosts

| Change | Commit |
|:---|:---|
| `rndis=off`; `usb-net` migration; tablet protocol | `7aebb77a43`, `bf68b11a71`, `cc932f6a7f` |
| timer thread, ARM timer and SMI latches, vsync generator | `d89e9bfb61` |
| compare clamp added, then undone; `post_load` re-arm | `5a39b9150e`, `2a357a4a69`, `65c941595b` |
| DMA 2D transfers of any width, rows whole | `7f4740ddc8` |
| hybrid-core placement, Windows then macOS | `e7699165ab`, `b46fa9765d` |
| debug-walk domain check, self-declared unnecessary | `1dd95655a9` |

<!-- doccrate:keep-together:end -->


