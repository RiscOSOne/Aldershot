# 8. The bill

The approach made RISC OS on an emulated Pi 4 possible, and in several places
made it fast. It also cost things, and the project's records are candid about
every one of them. This chapter collects them.

## 1. Silent failure is the characteristic failure mode

A hardware model that is wrong usually fails loudly — a bus error, a hang with a
register named. A substitute that is missing or unconfigured often fails by doing
nothing, and nothing looks the same as "not implemented".

**Sound on Windows was silent for two days.** Sound needs two command-line
options to reach the host audio back end. Without them it *"fails quietly"*, and
silence *"is indistinguishable from sound never having been built."* The lesson
recorded:

> *a feature is not ported until the launcher that reaches it is ported too*

The same shape appears elsewhere: v0 HostFS's identity-mapping fallback corrupted
memory silently, and `*SetType` once succeeded without doing anything. The
response in each case was to make the failure loud — refuse to load, return an
error, name the offset.

## 2. The guest has to carry modules, and modules bring their own problems

Every paravirtual substitute needs a RISC OS module in the guest. That buys
correctness through fall-through, and it creates a class of failure that pure
emulation never has.


<!-- doccrate:keep-together:start -->

### Four ways a module fails

| problem | what happened |
|:---|:---|
| **vector ordering** | GVFill spliced into the ROM *"never sees a sprite plot"*, because a module later in start-up claims the sprite vector first |
| **ROM-safety** | a module in ROM may not write to its own image; claims must pass the private word; module sizes go in a different register; the C runtime relocates in place |
| **shadowing** | loading HostFS from a card's boot sequence replaced the ROM's version 2.00 with an older 1.01 |
| **version coupling** | a v1 HostFS module refuses a v0 emulator — deliberately, but it couples releases |

<!-- doccrate:keep-together:end -->


ROM-safety in particular became its own body of knowledge, recorded as a recipe
because it was learned one failure at a time.

## 3. Acceleration puts correctness at risk

Fall-through protects correctness only as long as what *is* accelerated is
exactly right, and that took several attempts:

- masked sprites *"drew the desktop wrong"* — 484,513 differing pixels — and were
  backed out
- an unexplained plot-action bit changed 3,468 pixels
- a Windows run found 2,688 differing bytes where the Mac found none
- an origin and clipping slip put every host-drawn sprite in the wrong place

Each was caught because the project compared host-drawn output against ROM-drawn
output pixel for pixel. Without that comparison, several would have shipped.

## 4. Coverage depends on configuration

The blitter accepts only the pixel format it understands. At one Windows
machine's default colour depth, the desktop scrolled with *zero* blitter
operations. A cache of screen-mode constants read from the wrong place took none
of 70 plots. The design is still correct in those cases — it just is not doing
anything.

## 5. Fidelity

- **The pointer is gone from screen dumps**, because the host composites it —
  matching real hardware, but surprising the tooling.
- **Intermediate redraw states became visible.** Reading guest RAM on the host's
  clock shows states a real Pi displays for one scan-out. That reversed an earlier
  design note that *"a torn frame is exactly what a real monitor shows."*
- **Datestamps were an hour off under British Summer Time.** The fix added the
  host's UTC offset, on the reasoning that RISC OS keeps local time in a
  filestamp. A later design note argues the opposite — that RISC OS datestamps are
  UTC by definition, and the right fix is UTC on the wire with the time zone set in
  the guest. The code has not followed the note yet, so the question is open.

## 6. Portability to real hardware is nil for the paravirtual parts

The blitter *"models no real BCM2711 hardware"*. On a real Pi, GVFill and HostFS
2.x have nothing to talk to; they report that there is no host blitter or no
doorbell and stand aside. That is by design, and it is still a cost: the
substitutes are an emulator-only layer, and nothing about them helps RISC OS on
silicon.

## 7. Assumptions that hold only under TCG

Writing guest RAM from the host, behind the CPU's back, is safe today because TCG
has no cache model. The record flags the assumption explicitly:

> *If an HVF or KVM path ever appears, this needs revisiting.*

That is the other side of chapter 7's argument. Hardware virtualisation would
make the doorbell far more valuable — and would invalidate some of the shortcuts
that made the doorbell simple.

## 8. Snapshots and state outside the device model

- host-thread timer deadlines were not restored until a post-load hook re-armed
  them
- the doorbell device keeps no migration state, while the blitter migrates its
  registers
- *"A snapshot decides its own ROM"* — a machine saved without HostFS comes back
  without it

## 9. Namespaces

The filing-system number HostFS uses is not allocated by RISC OS's maintainers,
and host file names that Windows cannot store needed a private-use character
mapping.

## 10. The records contradict themselves in places

A walkthrough built on design notes has to check them against each other, and
these disagree in several places:


<!-- doccrate:keep-together:start -->

| claim | the problem |
|:---|:---|
| blitter covers 98.2% of sprite pixels | measured on the masked-sprite build that was backed out; the current figure is **94.3%** |
| the relocatable module area is identity-mapped | one file says it is, another says it is not; HostFS translates addresses rather than assuming |

<!-- doccrate:keep-together:end -->


<!-- doccrate:keep-together:start -->

#### The contradictions, continued


| claim | the problem |
|:---|:---|
| which peripheral window can hold a doorbell | one comment calls a region unreachable, another calls it *"provably live"* and uses it |
| the VCHIQ page-list format | a comment still describes the 12-bit form; the code uses the 36-bit one |
| HostFS module size target of 250–300 lines | the module is several times that |
| HostFS is whole-file only | an older README; streams and buffered I/O have since landed |

<!-- doccrate:keep-together:end -->


None is a defect in the system. All of them are the ordinary drift of notes
written at speed, and they are why every figure in this document was checked
against its commit.

## The method that kept it honest

Every item above was found, measured and recorded rather than discovered by a
user. The records name the discipline that did it, and it is the same one that
found the first blocker in chapter 1:

> *Every wrong turn this project has taken came from writing code before taking
> the measurement.*

> *Tracing beat theorising every time.*

And one refinement specific to measuring speed: alternate the variants being
compared — A, B, A, B — rather than running one then the other, so a warming host
cannot flatter whichever ran second.


<!-- doccrate:keep-together:start -->

## Things that will bite

| if you… | …this happens |
|:---|:---|
| add a substitute without making its absence loud | it fails silently, and looks unimplemented |
| port a feature but not the launcher flags that reach it | the feature does nothing on that host |
| accelerate a drawing case without a pixel comparison | a desktop that is subtly wrong |

<!-- doccrate:keep-together:end -->


*Things that will bite, continued:*


<!-- doccrate:keep-together:start -->

| if you… | …this happens |
|:---|:---|
| splice a vector-claiming module into the ROM | a later module may claim the vector first |
| write a ROM module like a soft-loaded one | ROM-safety failures, often late and obscure |
| assume guest memory is mapped | lazily mapped application space reads as absent |
| enable hardware virtualisation later | the host-writes-guest-RAM shortcuts need revisiting |
| trust a figure in the README | check it against the commit that measured it |

<!-- doccrate:keep-together:end -->


