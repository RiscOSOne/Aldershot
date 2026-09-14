# Aldershot

**RISC OS, on the computer you already own.**

This is the front door. It is where you find out what exists, whether it
works yet, and where to get it. There is no software in this repository
itself — just these pages and, when things are ready, the downloads.

---

## What's here

| | | |
| --- | --- | --- |
| 🍎 | **[RISC OS for the Mac](#risc-os-for-the-mac)** | Works. **[Download RISCOSQEA72v1](https://github.com/albanread/Aldershot/releases/tag/RISCOSQEA72v1)** for Apple silicon · [User guide](mac/user-guide.md) |
| 🪟 | **[RISC OS for Windows](#risc-os-for-windows)** | Works. No download yet — see below. |

More will be added as they become worth your time. Something is listed
here when it actually runs, not when it is started.

Curious how it works? The [developer walkthroughs](walkthroughs/README.md)
tell the whole story, from the first boot in QEMU to the files on your disc.

---

## RISC OS for the Mac

RISC OS 5.30, running on your Mac in its own window. The desktop, the
icon bar, the Filer, NetSurf — the real thing, not a picture of it.

It works by pretending to be a Raspberry Pi, which is the machine RISC OS
is built for these days. You do not have to know or care about that; you
open the app and RISC OS starts up.

> **[Download RISCOSQEA72v1 for Apple silicon](https://github.com/albanread/Aldershot/releases/tag/RISCOSQEA72v1)** · [User guide](mac/user-guide.md)

### What you can do with it

- **Use the desktop** — windows, menus, the icon bar, the lot
- **Browse the web** with NetSurf, over your Mac's internet connection
- **Share files with your Mac** — the RISC OS disc is a folder on your
  Mac, one you choose the first time. Drop a file in there on the Mac
  and it's inside RISC OS; save something in RISC OS and it's there on
  the Mac. This is the thing emulators usually make hard.
- **Hear it** — sound comes out of your Mac's speakers
- **Type and click** normally
- **Fill your screen** — see below

It takes about **twenty seconds** from starting the app to a desktop you
can use.

### Screen sizes

RISC OS is not stuck at the small sizes an emulator usually gives you. It
offers the same list a real monitor would, and you pick from RISC OS's own
Display Manager while it is running:

640×480 · 800×600 · 1024×768 · 1280×720 · 1280×800 · 1280×1024 ·
1440×900 · 1600×1200 · 1920×1080 · **1920×1200**

The last two are the interesting ones on a Mac. **1920×1200** is 16:10,
which is the shape of a MacBook screen, so a RISC OS desktop fills it
properly rather than sitting in a letterbox — and on a Retina display it
lands one RISC OS pixel to one screen pixel, which is about as sharp as
it gets.

You can also start up in a particular size rather than picking one each
time. The window opens to match whatever RISC OS is running, and you can
resize it freely afterwards — the picture scales to fit, so a big desktop
in a small window still works.

### The three mouse buttons

RISC OS expects a three-button mouse, and Macs have not shipped one for a
very long time. So:

| To press | Do this |
| --- | --- |
| **Select** (the normal one) | Click |
| **Menu** (opens menus — you'll want this a lot) | **Control**-click |
| **Adjust** | **Command**-click |

Option-click and Shift-click also give you Adjust, if either is comfier.

### What you'll need

**A Mac with Apple silicon** — an M1 or newer — running **macOS 26** or
later.

That's it. RISC OS itself is in the download:

- **The ROM** — RISC OS 5.30 from [RISC OS Open](https://www.riscosopen.org),
  the people who make RISC OS. It is their Raspberry Pi release, published
  under the Apache 2.0 licence, with our HostFS filing system added so
  that RISC OS can see the disc below.
- **A minimal guest file system** — the disc RISC OS boots from: a
  cut-down version of RISC OS Open's own disc, with the desktop and its
  Configure tools, NetSurf, StrongED, PipeDream, Ovation Pro and a shelf
  of utilities. No developer tools, no app store, no manuals, no games —
  enough to use, small enough to download.

There is nothing to fetch from anywhere else and nothing to set up.

### Installing it

Open the disk image and drag **RISCOSQEA72v1** to Applications, then
open it. It is signed and notarised, so macOS only asks, the first time,
whether you're sure about an app you downloaded.

On its first run the app asks where to keep the RISC OS disc: a folder
called **RISCOS** in your home folder, or one you choose. That folder is
your disc from then on; a later version of the app never replaces it,
and holding down **Option** as the app starts lets you choose again.

The [user guide](mac/user-guide.md) covers the rest: the mouse and
keyboard, screen sizes, how files cross between the Mac and RISC OS, and
what to do when something goes wrong.

### Before you get excited

**This is a first release** — RISCOSQEA72v1. It runs, and it runs well,
but it hasn't had the polish a finished thing deserves. A few honest
notes:

- It needs macOS 26. Older systems will refuse to run it, for now.
- The volume control inside RISC OS doesn't do anything — use your Mac's.
- The first few seconds of startup are slower than they ought to be.
- On a high-resolution screen the window opens small — one RISC OS pixel
  to one screen pixel. Drag it bigger; the picture scales.

It's a young project. It works, and it's genuinely usable. If that
sounds fine to you, you'll get on with it well.

---

## RISC OS for Windows

RISC OS 5.30, running on your PC in its own window. The desktop, the
icon bar, the Filer, NetSurf — the real thing, not a picture of it.

It works the same way the Mac one does: by pretending to be a
Raspberry Pi, which is the machine RISC OS is built for these days.
You do not have to know or care about that; you start it up and RISC
OS boots.

### What you can do with it

- **Use the desktop** — windows, menus, the icon bar, the lot
- **Browse the web** with NetSurf, over your PC's internet connection
- **Share a folder with RISC OS** — a folder on your Windows drive
  shows up inside RISC OS as a disc called HostFS. Copy files both
  ways: drop a file in the folder on Windows, and it's there in RISC
  OS; save one in RISC OS, and it appears in the folder. This one is
  worth knowing about, because it's the thing emulators usually make
  hard.
- **Save the whole machine** — take a snapshot from the window menu,
  and later pick up exactly where you left off, in under a second
  instead of waiting for a boot
- **Take screenshots** with the Print Screen key
- **Type and click** normally
- **Fill your screen** — see below

It takes about **a minute and a half** from starting it to a desktop
you can use. Yes, the Mac one says twenty seconds; yes, that is
annoying. A snapshot resume is under a second, which takes the sting
out of it.

### Screen sizes

RISC OS is not stuck at the small sizes an emulator usually gives you.
It offers the same list a real monitor would, and you pick from RISC OS's
own Display Manager while it's running:

640×480 · 800×600 · 1024×768 · 1280×720 · 1280×800 · 1280×1024 ·
1440×900 · 1600×1200 · 1920×1080 · **1920×1200**

You can also start up in a particular size rather than picking one each
time. The window opens to match whatever RISC OS is running, and you can
resize it freely afterwards — the picture scales to fit, so a big
desktop in a small window still works and stays readable.

### The three mouse buttons

RISC OS expects a three-button mouse. If you have a three-button mouse,
it just works. If you don't:

| To press | Do this |
| --- | --- |
| **Select** (the normal one) | Left-click |
| **Menu** (opens menus — you'll want this a lot) | **Middle**-click, or **Shift**+F10 |
| **Adjust** | **Right**-click |

While you're in a menu, the pointer stays where RISC OS put it. If the
pointer ever gets out of step with your mouse, middle-click or press
**Ctrl+Alt+G** to grab it; same keys to let go.

### What you'll need

**A PC running Windows 10 or 11**, 64-bit.

**RISC OS itself**, which you download once, free, from the people who
make it: [RISC OS Open](https://www.riscosopen.org/content/downloads/raspberry-pi).
You want the **RPi ROM stable** download and the **RISC OS Pi** one.

The Windows download doesn't yet include RISC OS — the Mac one does,
see above — so you fetch it once from RISC OS Open. It's free and it
takes a minute.

### One thing to know before you use it

**Close the window to switch off.** The window's close button shuts
RISC OS down properly, like switching off a real machine. Please don't
end it from Task Manager — that's yanking the plug out mid-write, and
discs (real or emulated) don't like it.

### Before you get excited

**There is no download yet.** It runs, and it runs well, but it hasn't
been packaged up into something you can double-click and install.
That's coming. Right now you'd have to build it yourself, which is a
job for someone comfortable with a command line.

A few other honest notes:

- The first boot is slower than every boot after it — and every boot
  is slower than it ought to be. Snapshots are the cure.
- There's no sound yet. That's coming.
- You can't drag files from Windows into RISC OS by dropping them on
  the window — use the shared folder instead (above), which is better
  anyway.

It's a young project, same as the Mac one. It works, and it's usable,
but it hasn't had the polish a finished thing deserves.

---

## Downloads

- **Mac, Apple silicon:** [RISCOSQEA72v1](https://github.com/albanread/Aldershot/releases/tag/RISCOSQEA72v1), a disk image. Open it
  and drag the app to Applications. Read the [user guide](mac/user-guide.md).
- **Windows:** nothing to download yet.

Every release is on this repository's
[Releases](https://github.com/albanread/Aldershot/releases) page.

---

## For developers

The [developer walkthroughs](walkthroughs/README.md) explain how all of
this was built, for anyone who wants to understand it or work on it:

- [RISC OS on a Pi 4, in QEMU](walkthroughs/QemuA72Walkthrough/index.md) — start here
- [Fake it in software](walkthroughs/FakeItInSoftwareWalkthrough/index.md)
- [Graphics and sound, done by the host](walkthroughs/GraphicsSoundWalkthrough/index.md)
- [HostFS: a host directory as a RISC OS disc](walkthroughs/HostFSWalkthrough/index.md)
- [Mojo for RISC OS](walkthroughs/MojoRISCOSWalkthrough/index.md)

Each one is also a single PDF.

---

## The small print

The software is free and open source, under the
[GNU General Public License, version 2](LICENSE) — the same licence QEMU
uses, which the Mac application is built from.

Each project also keeps its own code in its own place. For the Mac and
Windows ones that's [RISCOSQEMUA72](https://github.com/albanread/RISCOSQEMUA72).
The [walkthroughs](walkthroughs/README.md) are a long and fairly candid
account of how they were built, if you're curious about that sort of thing.

**RISC OS is included in the Mac download**: RISC OS 5.30 as published
by RISC OS Open Ltd under the Apache 2.0 licence, with our HostFS module
added, plus a minimal guest file system drawn from their disc image. The
applications on that disc belong to their authors and keep their own
licences. The Windows download does not include RISC OS; you get it
from RISC OS Open.

## Standing on

The Mac application is built on **[QEMU](https://www.qemu.org)**, the
machine emulator that does the hard part — pretending to be a Raspberry
Pi convincingly enough that RISC OS never notices. Our thanks to everyone
who built it.

**[RISC OS Open Ltd](https://www.riscosopen.org)** make RISC OS 5 and
publish it as open source. The ROM and the disc contents in the Mac
download are theirs.

**The source for what we release is here:**

- **[albanread/RISCOSQEMUA72](https://github.com/albanread/RISCOSQEMUA72)**
  — our fork, branch `riscos-pi4`. This is the one. Everything that makes
  RISC OS run, and the Mac and Windows applications themselves, live here.
- [gitlab.com/qemu-project/qemu](https://gitlab.com/qemu-project/qemu) —
  upstream QEMU, which the fork is based on.
