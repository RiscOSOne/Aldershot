# 8. Shipping it

A working module and device are not yet something a user can switch on. This chapter
covers the two ways HostNet reaches a running machine: the developer launcher's
switch, which rebuilds the ROM, and the Windows release's switch, which is simply
where a file sits on the disc. It walks through the release launcher, the HostNet
item in the window menu, the one hazard both have to avoid, the installer's rules,
and the start-up file that serves both network stacks. It ends with the Mac, where
the work is not yet committed.

## Two ways to reach a network


<!-- doccrate:keep-together:start -->

### The machines and their stacks

| Setup | Network card | ROM networking | Doorbell | Who serves sockets |
|:---|:---|:---|:---|:---|
| `run.py`, the default | slirp and `usb-net` | running | off | the ROM's Internet 5.67 |
| `run.py --hostnet` | none | unplugged: 98, 106, 107, 108 | on | HostNet, spliced into the ROM |
| Windows release, HostNet on | attached, unused | Internet replaced as HostNet loads | on | HostNet, loaded from the disc |
| Windows release, HostNet off | slirp and `usb-net` | running | off | the ROM's Internet 5.67 |
| Mac release and launchers | slirp and `usb-net` | running | off | the ROM's Internet 5.67 |

<!-- doccrate:keep-together:end -->


<!-- doccrate:keep-together:start -->

## The developer's switch

`run.py`, the Windows developer launcher, takes `--hostnet`. The option splices
HostNet into the ROM, unplugs the ROM's network modules in the CMOS it boots with,
turns the doorbell on, and attaches no network card:

```python
def command_line(args, overlay):
    # Two ways to reach a network, and never both.  With --hostnet the
    # guest has no stack to drive a NIC with, so attaching one would be an
    # emulated card nothing ever opens; without it, slirp and usb-net as
    # before.  The doorbell is off unless asked for, so a machine that has
    # not been switched over is untouched.
    net = (["-global", "hostnet.sockets=on"] if args.hostnet else
           ["-netdev", "user,id=n0,domainname=lan",
            # domainname: RISC OS asks for option 15 in its parameter
            # list.  It does not fix DHCP, but it is one less thing the
            # guest asked for and did not get.
            "-device", "usb-net,netdev=n0,rndis=off,bus=usb-bus.0,port=1.3"])
```

<!-- doccrate:keep-together:end -->


This route rebuilds the ROM image and the CMOS blob with Python tools at launch. That
suits a development machine and would not suit an end user's, which is why the
release took the other route.

## The Windows release: the module's folder

The Windows release keeps the ROM stock. HostNet ships as a module file on the disc,
which is also the HostFS share, and the folder it is in is the switch:


<!-- doccrate:keep-together:start -->

### The switch is a file

| Where the file is | State | What happens at boot |
|:---|:---|:---|
| `Modules\HostNet,ffa` | on | `!Boot` loads it; titled Internet, it replaces the ROM's Internet module |
| `Modules\Disabled\HostNet,ffa` | off | nothing loads it; the ROM's stack drives the network card |

<!-- doccrate:keep-together:end -->


The loader is a one-line boot command file, `PreDesk.HostModules`, which the HostFS
work added to `!Boot`:

```
IfThere HostFS:$.Modules Then Repeat RMLoad HostFS:$.Modules -Type &FFA -Sort -Continue
```

It loads every module file in `Modules`, in name order, and carries on past one that
fails. It does not descend into subfolders, so a module in `Modules\Disabled` is never
loaded. `make-release.py`'s `place_hostnet` puts the built module in one folder or the
other, according to `--hostnet on` or `off`; on is the default.

## The launcher

The Windows launcher is a small compiled program. For HostNet it does two things,
described at length in its header comment:

- **It lights the doorbell only when HostNet is on.** Off is the device's default, and
  the machine then looks exactly as it did before HostNet.
- **It attaches the network card either way.** Under HostNet nothing uses the card.
  But if the user switches HostNet off and restarts RISC OS without restarting the
  emulator, the ROM's stack needs a card, or it boots with "Route: Network is
  unreachable".


<!-- doccrate:keep-together:start -->

#### The file test

The code is a file test:

```c
    /* The doorbell, lit only with HostNet on: its module in Modules itself.
     * The card is attached regardless, below; see the top of the file. */
    _snwprintf(hostnet, MAX_PATH, L"%ls\\Modules\\HostNet,ffa", disc);
    attrs = GetFileAttributesW(hostnet);
    net = (attrs != INVALID_FILE_ATTRIBUTES
           && !(attrs & FILE_ATTRIBUTE_DIRECTORY))
        ? L"-global hostnet.sockets=on " : L"";
```

<!-- doccrate:keep-together:end -->


## The hazard: a dark doorbell

Both switches share one way to leave a machine with no network stack at all, and the
comment on the menu code spells it out. When a module titled Internet loads, the
kernel kills the existing Internet module first, and only then runs the newcomer's
initialisation. If the doorbell is dark at that moment, HostNet declines to load, as
chapter 7 described, and there is no Internet module left.


<!-- doccrate:keep-together:start -->

### What prevents it

| Situation | Doorbell | What the design does |
|:---|:---|:---|
| emulator started with HostNet on | lit by the launcher | nothing more needed |
| switched on from the menu, then RISC OS restarted | lit by the menu at the moment of switching | HostNet finds the doorbell and loads |
| switched off from the menu | left lit for this session | a HostNet already running keeps serving its sockets |
| file moved by hand to `Modules` mid-session | dark | the menu warns: on "when you quit and start again" |

<!-- doccrate:keep-together:end -->


<!-- doccrate:keep-together:start -->

## The window menu

The Direct3D window's system menu gains a **HostNet** item, rebuilt every time the
menu opens. It appears only on a machine with a HostFS share whose `Modules` or
`Modules\Disabled` folder holds the module, and it is ticked when the module is in
`Modules`. Choosing it moves the file:

```c
    disabled = g_build_filename(dir, "Disabled", NULL);
    from = g_build_filename(on ? disabled : dir, DX11_HOSTNET_LEAF, NULL);
    to = g_build_filename(on ? dir : disabled, DX11_HOSTNET_LEAF, NULL);

    if (!g_file_test(from, G_FILE_TEST_IS_REGULAR)) {
        g_snprintf(why, why_len, "%s is not there any more.", from);
        return -1;
    }
    if (g_mkdir_with_parents(on ? dir : disabled, 0755) != 0
        || (g_file_test(to, G_FILE_TEST_EXISTS) && g_remove(to) != 0)
        || g_rename(from, to) != 0) {
        g_snprintf(why, why_len, "Cannot move %s to %s: %s", from, to,
                   g_strerror(errno));
        return -1;
    }
    if (on) {
        bql_lock();
        hn->enabled = true;
        bql_unlock();
    }
    return 0;
```

<!-- doccrate:keep-together:end -->


The paths are built with GLib from the share's root, as HostFS builds them, so the
menu and the filing system agree about where `Modules` is. Switching on also lights
the doorbell, under QEMU's global lock. Nothing ever darkens it.


<!-- doccrate:keep-together:start -->

### The menu's notes

The menu knows three things: where the file is, whether the doorbell is lit, and
whether the running RISC OS has rung it since it started. When the tick and the
running machine disagree, a greyed note says so:

| File in | Lit | Rung | Note under the item |
|:---|:---|:---|:---|
| `Modules` | yes | yes | none: on, and running |
| `Modules` | yes | no | "On when RISC OS next starts" |
| `Modules` | no | no | "On when you quit and start again" |
| `Disabled` | any | yes | "Off when RISC OS next starts" |
| `Disabled` | any | no | none: off |

<!-- doccrate:keep-together:end -->


The "rung" flag is the device's `rung` field from chapter 3. The module rings fifty
times a second, so within one tick of HostNet loading the menu knows that HostNet is
what this boot is running, even after the file has been moved.


<!-- doccrate:keep-together:start -->

## The installer

The Windows installer, built with Inno Setup, installs the disc into the user's
profile folder, and never overwrites a file already there. HostNet is the exception,
with its own entry and two rules, written in the installer's Pascal script:

```pascal
{ Where HostNet's module goes: where it already is, or for a new machine
  where the release starts it, @HOSTNET_DIR@. }
function HostNetDir(Param: String): String;
begin
  if FileExists(DiscDir() + '\Modules\HostNet,ffa') then
    Result := DiscDir() + '\Modules'
  else if FileExists(DiscDir() + '\Modules\Disabled\HostNet,ffa') then
    Result := DiscDir() + '\Modules\Disabled'
  else
    Result := DiscDir() + '\@HOSTNET_DIR@';
end;
```

<!-- doccrate:keep-together:end -->


<!-- doccrate:keep-together:start -->

### The two rules

| Rule | Why |
|:---|:---|
| the module is always replaced, in whichever folder the user left it | the module and the emulator it talks to are one version, and a reinstall must never flip the switch |
| a disc from before HostNet does not get the module at all | that disc's start-up file was written for the network card alone, and fails under HostNet before the name servers are set |

<!-- doccrate:keep-together:end -->


The second rule is `HostNetWanted`: a new machine gets the module, and so does one
that already has it in either folder. An older machine gets neither the module nor,
therefore, the menu item that would switch it on.

## One start-up file for two stacks

The disc's `Choices:Internet.Startup` runs whichever stack is the Internet module. The
network card needs interfaces, an address and a route configured; under HostNet those
commands fail, because there is no interface, and the `CheckError` after them would
stop `!Internet`'s start-up before the file that sets the name servers.


<!-- doccrate:keep-together:start -->

#### The shared start-up file

`e8bdae8d8d` therefore splits the file in two. `make-release.py` writes a `Startup`
that holds only what both stacks need, and hands the card's configuration to a second
file when the stack is the ROM's:

```
Set Inet$HostName RISCOSpi

| Read by !Run after this file returns: no gateway, no RouteD.
Set Inet$IsGateway ""
Set Inet$RouteDOptions ""

| The interfaces, only when the Internet module is older than HostNet.
RMEnsure Internet 6.00 Run Choices:Internet.Interfaces
```

<!-- doccrate:keep-together:end -->


`RMEnsure` runs its command when the named module is older than the version given.
HostNet is Internet 6.00, so under HostNet the interfaces file is skipped; the ROM's
Internet 5.67 is older, so with HostNet off it runs. `Choices:Internet.Interfaces`
holds the card's configuration: the fixed slirp address from `d0de1c013c`, the route,
loopback and the file-sharing modules. The name servers stay in the `User` file,
which both stacks run.

A banner at the top of `Startup` warns that saving from `!InetSetup` replaces the file
with one that "knows nothing of HostNet".

## What the public site tells users

The fork's public site describes the Windows release in plain terms. Paraphrased:
HostNet redirects RISC OS's IP sockets to Windows, which makes the connections, and
it should be faster and more reliable than the emulated USB network. It is not a
complete network stack. It serves TCP and UDP programs, such as browsing, name
lookups, fetching files and programs that listen, but tools like `Ping`, `IfConfig`,
`ARP`, `route` and `InetStat` have nothing to report, and ShareFS, Access and Econet do
not work over it. It is IPv4 only, on at install, switched from the window menu, and a
switch takes effect when RISC OS next starts.

## The Mac

At `d535505fad`, no committed Mac launcher knows about HostNet. The Mac app and the Mac
development scripts still attach slirp and `usb-net`, and the only committed Mac
change is the device build fix, `20bf267db5`. A complete Mac integration existed in
the working tree on 15 September, uncommitted, and follows a different model from the
Windows release: the ROM splice and CMOS unplug, with no card. Chapter 9 describes it.
