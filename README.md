# Aldershot

**RISC OS, on the computer you already own.**

This is the front door. It is where you find out what exists, whether it
works yet, and where to get it. There is no software in this repository
itself — just these pages and, when things are ready, the downloads.

---

## What's here

| | | |
| --- | --- | --- |
| 🍎 | **[RISC OS for the Mac](#risc-os-for-the-mac)** | Works. No download yet — see below. |
| 🪟 | **[RISC OS for Windows](#risc-os-for-windows)** | Works. No download yet — see below. |

More will be added as they become worth your time. Something is listed
here when it actually runs, not when it is started.

---

## RISC OS for the Mac

RISC OS 5.30, running on your Mac in its own window. The desktop, the
icon bar, the Filer, NetSurf — the real thing, not a picture of it.

It works by pretending to be a Raspberry Pi, which is the machine RISC OS
is built for these days. You do not have to know or care about that; you
open the app and RISC OS starts up.

### What you can do with it

- **Use the desktop** — windows, menus, the icon bar, the lot
- **Browse the web** with NetSurf, over your Mac's internet connection
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

**A Mac with Apple silicon** — an M1 or newer.

**RISC OS itself**, which you download once, free, from the people who
make it: [RISC OS Open](https://www.riscosopen.org/content/downloads/raspberry-pi).
You want the **RPi ROM stable** download and the **RISC OS Pi** one.

We can't include RISC OS in the download. It isn't ours to give away —
it's RISC OS Open's, and they'd rather you got it from them. It's free
and it takes a minute.

### Before you get excited

**There is no download yet.** It runs, and it runs well, but it hasn't
been packaged up into something you can double-click and install. That's
coming. Right now you'd have to build it yourself, which is a job for
someone comfortable with a command line.

A few other honest notes:

- You can't drag files from your Mac into RISC OS yet. Anything you want
  in there has to be on the disc image already.
- The volume control inside RISC OS doesn't do anything — use your Mac's.
- The first few seconds of startup are slower than they ought to be.

It's a young project. It works, and it's genuinely usable, but it hasn't
had the polish a finished thing deserves. If that sounds fine to you,
you'll get on with it well.

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
You want the **RPi ROM stable** download and the **RISC OS Pi** one —
the same downloads as the Mac.

We can't include RISC OS in the download. It isn't ours to give away —
it's RISC OS Open's, and they'd rather you got it from them. It's free
and it takes a minute.

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

Nothing to download yet. When there is, it'll be on this repository's
**Releases** page.

---

## The small print

The software is free and open source, under the
[GNU General Public License, version 2](LICENSE) — the same licence QEMU
uses, which the Mac application is built from.

Each project also keeps its own code in its own place. For the Mac and
Windows ones that's [RISCOSQEMUA72](https://github.com/albanread/RISCOSQEMUA72),
where you'll find a long and fairly candid account of how they were
built, if you're curious about that sort of thing.

**RISC OS itself is not distributed here**, in any form. It belongs to
RISC OS Open Ltd and you get it from them.

## Standing on

The Mac application is built on **[QEMU](https://www.qemu.org)**, the
machine emulator that does the hard part — pretending to be a Raspberry
Pi convincingly enough that RISC OS never notices. Our thanks to everyone
who built it.

**The source for what we release is here:**

- **[albanread/RISCOSQEMUA72](https://github.com/albanread/RISCOSQEMUA72)**
  — our fork, branch `riscos-pi4`. This is the one. Everything that makes
  RISC OS run, and the Mac and Windows applications themselves, live here.
- [gitlab.com/qemu-project/qemu](https://gitlab.com/qemu-project/qemu) —
  upstream QEMU, which the fork is based on.
