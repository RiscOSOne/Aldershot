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

It takes about **twenty seconds** from starting the app to a desktop you
can use.

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
- Changing the screen resolution while it's running doesn't work properly.
- The volume control inside RISC OS doesn't do anything — use your Mac's.
- The first few seconds of startup are slower than they ought to be.

It's a young project. It works, and it's genuinely usable, but it hasn't
had the polish a finished thing deserves. If that sounds fine to you,
you'll get on with it well.

---

## Downloads

Nothing to download yet. When there is, it'll be on this repository's
**Releases** page.

---

## The small print

The software is free and open source, under the
[GNU General Public License, version 2](LICENSE) — the same licence QEMU
uses, which the Mac application is built from.

Each project also keeps its own code in its own place. For the Mac one
that's [RISCOSQEMUA72](https://github.com/albanread/RISCOSQEMUA72), where
you'll find a long and fairly candid account of how it was built, if
you're curious about that sort of thing.

**RISC OS itself is not distributed here**, in any form. It belongs to
RISC OS Open Ltd and you get it from them.
