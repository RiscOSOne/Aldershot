# Developer walkthroughs

How RISC OS came to run on the Mac and on Windows, written for developers
who want to know how it works, or to work on it. Each walkthrough reads
chapter by chapter here on GitHub, and each has a PDF of the whole thing.

| Walkthrough | What it covers | Read |
| --- | --- | --- |
| **RISC OS on a Pi 4, in QEMU** | How stock QEMU was taught to boot the stock RISC OS 5.30 Raspberry Pi ROM on an emulated Raspberry Pi 4 whose Cortex-A72 runs in 32-bit mode, and why almost none of the work was about the CPU. | [Chapters](QemuA72Walkthrough/index.md) · [PDF](QemuA72Walkthrough/QemuA72-Walkthrough.pdf) |
| **Fake it in software** | Why RISC OS boots on an emulated Pi 4 while almost none of the Pi's hardware is modelled: the emulator answers the operating system's questions instead. | [Chapters](FakeItInSoftwareWalkthrough/index.md) · [PDF](FakeItInSoftwareWalkthrough/FakeItInSoftware-Walkthrough.pdf) |
| **Graphics and sound, done by the host** | How the desktop reaches a window on Windows and on the Mac, how drawing RISC OS does in ARM code moves onto the host, and how sound plays, with no GPU or audio chip emulated. | [Chapters](GraphicsSoundWalkthrough/index.md) · [PDF](GraphicsSoundWalkthrough/GraphicsSound-Walkthrough.pdf) |
| **HostFS: a host directory as a RISC OS disc** | How a folder on the Mac or the PC became a RISC OS filing system, and then the disc the machine boots from, with no SD card at all. | [Chapters](HostFSWalkthrough/index.md) · [PDF](HostFSWalkthrough/HostFS-Walkthrough.pdf) |
| **Mojo for RISC OS** | How a fork of the Mojo compiler, a small Rust linker called roscc, and a C runtime put Mojo programs on RISC OS 5, first on an emulated StrongARM and then on the emulated Pi 4. | [Chapters](MojoRISCOSWalkthrough/index.md) · [PDF](MojoRISCOSWalkthrough/MojoRISCOS-Walkthrough.pdf) |

Start with **RISC OS on a Pi 4, in QEMU**: the others build on it.

The source they describe is
[albanread/RISCOSQEMUA72](https://github.com/albanread/RISCOSQEMUA72),
branch `riscos-pi4`.
