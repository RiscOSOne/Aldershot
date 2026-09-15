# 9. Proven, and what comes next

This chapter gathers what the commits prove, with the numbers, then the limits the
project states for users, and finally the work that was in progress on 15 September.

## What each step proved

Every sprint commit closes with a test run on the emulator:

<!-- doccrate:keep-together:start -->

#### The tests

| Step | Test | Result |
|:---|:---|:---|
| Sprint 0, `23fc9630c5` | boot with no network card; `*Modules`; `*HostNetPing`; start NetSurf | desktop reached; Internet at the spliced address, EtherUSB, EtherGENET and DHCP gone; ping answered; NetSurf idle with no error |
| Sprint 1, `0f73da70af` | `*GetHost www.riscosopen.org` through the stock Resolver | 91.203.57.12, in one exchange: a 30-byte query, a poll, a 94-byte answer |
| Sprint 1, failure path | the resolver pointed at an unreachable address | eight sends, then "Failed to look up", with the machine still running |
| Sprint 2, `89013ef0f2` | NetSurf over TLS | riscosopen.org in 1.3 s; Wikipedia's RISC OS article in 5.3 s over 27 sockets |

<!-- doccrate:keep-together:end -->

<!-- doccrate:keep-together:start -->

#### The tests, continued

| Step | Test | Result |
|:---|:---|:---|
| Sprint 3, `5b882f2a4a` | `listen.bas`, a guest server | 18 bytes echoed; seven retries in the blocking accept |
| Sprint 3 | `evtest.bas`, events only | `async`, then `broken`; eight bytes read on the event |
| `50ef1a1bcd` | registers, off and on | all zero off; `'HNET'`, 1, 1 on |
| `3c523600ea` | boot with the doorbell off | 138 modules, no Internet module, a clean desktop |
| `e8bdae8d8d` | a staged Windows release | both states boot and resolve; the menu moves the file |

<!-- doccrate:keep-together:end -->

<!-- doccrate:keep-together:start -->

## The numbers

| Measurement | Value | Source |
|:---|:---|:---|
| DHCP success on the old stack | 0 of 12 boots | `23fc9630c5` |
| USB transactions per 1500-byte frame, old path | 24 | `23fc9630c5` |
| desktop time, DHCP against static | 49 s against 28 s | `d0de1c013c` |
| NetSurf, riscosopen.org over TLS | 1.3 s; the old stack failed | `89013ef0f2` |
| NetSurf, Wikipedia's RISC OS article | 5.3 s, 27 sockets | `89013ef0f2` |

<!-- doccrate:keep-together:end -->

No throughput or latency figure exists for HostNet yet. Measuring throughput against
the old path is part of Sprint 5, which has not started.

<!-- doccrate:keep-together:start -->

## What HostNet does not do

These are the limits the code and the public notes state:

| Limit | Where stated |
|:---|:---|
| IPv4 only; `AF_INET6` answers `EAFNOSUPPORT` | `hn_creat`; the public site |
| no raw sockets or ICMP, so `Ping`, `IfConfig`, `ARP`, `route` and `InetStat` have nothing to report | the public site |
| ShareFS, Access and Econet do not work over it | the public site |
| a blocking call outside a TaskWindow holds the desktop until it completes or Escape is pressed | the module's wait loop comment |
| a restored snapshot has no open sockets | the saved-state comment |
| saving from `!InetSetup` replaces the start-up file that serves both stacks | the banner in `Startup` |

<!-- doccrate:keep-together:end -->

## Work in progress

The local checkout on 15 September held uncommitted work that brings HostNet to the
Mac, based on `3c523600ea`; this describes it as it stood. `build-hostnet.sh` finds
its tools on either host. `RISCOS_HOSTNET=1` makes the Mac scripts splice HostNet,
unplug the ROM stack and attach no card, and the Mac app ships it on by default, with
a small shell script that converts an older disc at launch. The Mac notes record a
run on 14 September: `*GetHost` resolving, NetSurf loading Wikipedia's RISC OS article
in 3.7 seconds, both BASIC tests passing, and 13 guest sockets counted by both the
trace and `lsof`.
