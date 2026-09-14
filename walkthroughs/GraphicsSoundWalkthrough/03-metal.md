# 3. The same program, another idiom

When the fork reached the Mac, the emulation needed no changes. What it needed was
a window. Commit `7a39b30571` added `ui/metal.{h,c,m}`: the Direct3D 11 front end
rebuilt in the Mac's own idiom, deliberately the same program. This chapter covers
what carried over and what changed. It also covers the one thing the Mac front end
grew that Windows has no twin for: an Apple Events surface designed for AI agents
to drive the machine.


<!-- doccrate:keep-together:start -->

## One program, two platforms

The Metal file's header states the design intent. Each frame, the UI thread pulls
the framebuffer view over a narrow C boundary, uploads the raw bytes, decodes them
with a per-format shader, and stretches the result over the whole view. Only the
platform mechanisms differ:

| Concern | Windows, `ui/dx11.*` | Mac, `ui/metal.*` |
|:---|:---|:---|
| language | C++17 behind `extern "C"` | Objective-C, no ARC, behind a C header |
| shaders | HLSL, compiled at start-up with `d3dcompiler` | Metal Shading Language, compiled with `newLibraryWithSource:` |
| per-format variants | preprocessor macros | Metal function constants: `BPP`, `SCALING`, `SCANLINES` |
| raw framebuffer | two dynamic byte buffers | a ring of three shared-storage buffers |

<!-- doccrate:keep-together:end -->


<!-- doccrate:keep-together:start -->

#### One program, two platforms, continued

| Concern | Windows, `ui/dx11.*` | Mac, `ui/metal.*` |
|:---|:---|:---|
| pacing | waitable swap chain | `CAMetalLayer` with display sync, plus an in-flight semaphore |
| thread hand-off | QEMU's `qemu_main` pointer | the same hand-off, which the Cocoa port introduced |

<!-- doccrate:keep-together:end -->


The Metal file includes no QEMU headers at all. It talks to the emulation only
through `ui/metal.h`, the twin of `ui/dx11.h`.

## Memory the GPU already sees

On Apple Silicon, CPU and GPU share memory. The Metal front end uses that directly:
the raw framebuffer bytes go into buffers created with shared storage, so the
upload is **a `memcpy` into memory the GPU already sees**:

```objc
/* The raw guest bytes, a ring deep enough that the frame the GPU is
 * still reading is never the one the CPU is writing.  Shared storage
 * means the "upload" is a memcpy into memory the GPU already sees --
 * on Apple silicon there is nothing further to transfer. */
for (i = 0; i < METAL_RING; i++) {
    fb.raw[i] = [m.device newBufferWithLength:bytes
                                      options:MTLResourceStorageModeShared];
    ...
}
```

The ring is three deep (`METAL_RING`). Metal command buffers complete
asynchronously, so the buffer the GPU is still reading must never be the one the
CPU is writing. A semaphore limits frames in flight. The decoded surface is a
private-storage texture at the guest's resolution: the scale pass's only input,
and what a screenshot reads back.

On an Intel Mac with a discrete GPU, "shared" memory implies a transfer across the
bus. The graphics design notes that its unified-memory reasoning may not hold
there. **INFERRED:** the per-frame cost on such a machine has not been measured.


<!-- doccrate:keep-together:start -->

## Mac input

A Mac mouse usually has one or two buttons, and RISC OS needs three. The front end
maps them with modifiers:

| Host action | RISC OS button |
|:---|:---|
| click | Select |
| Control-click | Menu |
| Command-, Option- or Shift-click | Adjust |
| a real middle button | Menu |

<!-- doccrate:keep-together:end -->


The modifier is withheld from the guest during the click, so Control-click does not
also send Control. The host cursor is hidden by a rule rather than by a grab: the
guest draws its own arrow exactly under the host's, so the host arrow is hidden
while the pointer is over the view in a key window. On Windows both arrows look
alike and nobody notices; on the Mac they do not.

One Mac pitfall is recorded so it is not repeated. An early build skipped drawing
while the window's `occlusionState` said it was hidden, and on the first run
nothing was ever drawn.

## An app, not a command line

A command-line QEMU is fine for a developer and awkward for everyone else. The Mac
port wraps the binary in an application bundle:

- `tools/make-bundle.sh` copies the binary, an `Info.plist`, the generated
  scripting dictionary and the icon into `RISCOSQEMU.app`, then ad-hoc signs and
  verifies it. The icon is original geometry.
- `tools/run-app.sh` launches it through `open`, with CoreAudio and no QMP socket.
  The script calls that the *user persona*.

## Apple Events, agents first

The Mac front end grew a control surface that Windows has no twin for. It is
designed, in the fork's own words, **for an AI agent as the first and primary
scripter**, with humans as a pleasant side effect. The design opens with a
principle modelled on the emulator's:

> rather than exposing QEMU, we should expose the functions needed to control and
> debug this machine … be as curated and typed as possible, and expose raw QEMU
> only if we are desperate. We do need a consent model though — the whole surface
> is running off trust.


<!-- doccrate:keep-together:start -->

### Why Apple Events rather than QMP

QMP already exposes everything. The design's case is about what an unattended agent
lacks with a bare socket:

| Need | QMP over TCP | Apple Events |
|:---|:---|:---|
| a client | one that speaks QMP's greeting and capabilities exchange | `osascript`, already on every Mac, including JavaScript for Automation |
| identity | a port number tied to one process | the bundle identifier, which survives restarts |
| consent | none | macOS asks the *sender* once, keyed to both identities |
| discoverability | read the QEMU docs | a machine-readable dictionary (sdef) of every command |

<!-- doccrate:keep-together:end -->


### How it is built

The commands live in one C table in `ui/metal.c`. The generator `mksdef.py` turns
that table into the scripting dictionary and a JSON description, so the interface
and its documentation cannot drift apart. There are 24 commands in suite `MQem`.
Each replies with a JSON envelope. Each is marked with one of two execution classes:

- **`fast`** runs synchronously under QEMU's big lock, on the UI thread, for reads
  and counters
- **`bh`** schedules a bottom half on QEMU's main loop and waits on a semaphore, for
  anything that changes the machine

Two rules keep the semaphore wait free of deadlock: the waiting UI thread holds no
lock, and a bottom half never calls back into the UI thread. Commands are
single-flight: a second caller gets `busy`.


<!-- doccrate:keep-together:start -->

### Measured

| Question | Result |
|:---|:---|
| Q1: are events delivered through the hand-run event pump? | yes, after one *Allow* click |
| Q2: latency | **16.7 ms** per warm Apple Event round trip, against **0.1 ms** for QMP |
| Q3: does consent survive a rebuild? | yes, once measured, for an ad-hoc-signed bundle |

<!-- doccrate:keep-together:end -->


The factor of about 160 settled the division of labour. Single commands for control
and inspection go over Apple Events. Chatty loops, such as input streams and
polling, stay on QMP.


<!-- doccrate:keep-together:start -->

### Two kinds of screenshot

The surface separates two questions that a single screenshot command would blur:

| Command | Returns | Time |
|:---|:---|:---|
| `screendump` | the raw guest framebuffer — *the truth, no decode* — without the pointer | 40 ms |
| `screenshot` | the decoded Metal surface, with the pointer sprite blended in | 72 ms |

<!-- doccrate:keep-together:end -->


A display bug shows up as a difference between the two.

### Lessons from building it

- **One instance, or the wires cross.** A relaunch while an old instance still ran
  left two apps with one bundle identifier. Apple Events reached one process while
  QMP answered from the other, and the surface looked hung.
- **Never walk the object tree per frame.** A crash sample showed the pointer read
  resolving the VCHIQ device through QEMU's whole object tree on *every* frame:
  milliseconds per frame, and a lock-free read of a tree other threads own. The
  device pointer is now cached once.
- **The standard `quit` event was a land mine.** It fell through to the default
  application terminate, with no disc write-back. It now takes the same clean
  power-off path as closing the window.

Breakpoints and single-step over Apple Events, planned as stage E4, are not built.

## The Intel Mac

The same code builds for Intel Macs, with no `arm64` conditionals. Chapter 7 of the
[QEMU A72 walkthrough](../QemuA72Walkthrough/07-hosts-and-whats-left.md) covers the
dependency build. The display acceptance test on a Xeon W-3235 compared a Metal
surface capture with a QMP screendump taken a second apart. They differed in **134
of 480,000 pixels**, a 12 × 22 block at dead centre: the pointer, caught mid-move.
Every other pixel was identical.
