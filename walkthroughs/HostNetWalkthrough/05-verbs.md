# 5. The verbs

The host device implements the socket calls one by one, and the sprints added them
in groups: UDP first, then TCP, then listeners and the message calls. This chapter
lists all 35 SWIs with the sprint that served each one. It then walks through the
translation layer that sits between a RISC OS program and the host's socket API:
socket addresses, message flags, socket options, ioctls and error numbers. It ends
with the limits the device imposes and the differences between Windows and POSIX
hosts.


<!-- doccrate:keep-together:start -->

## The dispatch

`hn_do_swi` is a single `switch` on the SWI number, grouped in the source by sprint.
The Sprint 1 group is where every other verb started:

```c
static void hn_do_swi(HostNetState *s, uint32_t swi, uint32_t *R, HNReply *r)
{
    switch (swi) {
    case HN_SWI_CREAT:      hn_creat(s, R, r); break;
    case HN_SWI_BIND:       hn_bind(s, R, r); break;
    case HN_SWI_SENDTO:     hn_sendto(s, R, r); break;
    ...
    /* Sprint 2: TCP. */
    case HN_SWI_CONNECT:    hn_connect(s, R, r); break;
    ...
    default:
        hn_errset(r, ROS_EOPNOTSUPP);
        break;
    }
}
```

<!-- doccrate:keep-together:end -->


Anything without a case answers `EOPNOTSUPP`, which the module turns into a RISC OS
error. That was the whole of Sprint 0: every call reached the host and came back
"not supported". Its commit points out that `ARP -s` reporting "ARP: socket" was
"the whole path working".

## All 35 SWIs

The SWI numbers are fixed by the Internet module's interface: &41200 plus the SWI's
position in this order, which is also the order of `swis.txt`:

> Creat, Bind, Listen, Accept, Connect, Recv, Recvfrom, Recvmsg, Send, Sendto,
> Sendmsg, Shutdown, Setsockopt, Getsockopt, Getpeername, Getsockname, Close,
> Select, Ioctl, Read, Write, Stat, Readv, Writev, Gettsize, Sendtosm, Sysctl,
> Accept_1, Recvfrom_1, Recvmsg_1, Sendmsg_1, Getpeername_1, Getsockname_1,
> InternalLookup, Version

So `Socket_Creat` is &41200, `Socket_Close` is &41210 and `Socket_Version` is &41222.
The six calls without `_1` that also have a `_1` form are the Internet 4 versions.


<!-- doccrate:keep-together:start -->

#### Served since

| Sprint | SWIs |
|:---|:---|
| 1 | `Creat`, `Bind`, `Sendto`, `Recvfrom`, `Recvfrom_1`, `Close`, `Ioctl`, `Select` for reading, `Gettsize`, `Version`, `Sysctl` |
| 2 | `Connect`, `Send`, `Recv`, `Read`, `Write`, `Readv`, `Writev`, `Shutdown`, `Setsockopt`, `Getsockopt`, the four name calls, `Select` in full |
| 3 | `Listen`, `Accept`, `Accept_1`, `Sendmsg`, `Sendmsg_1`, `Recvmsg`, `Recvmsg_1` |
| never | `Stat`, `Sendtosm`, `InternalLookup`: `EOPNOTSUPP` |

<!-- doccrate:keep-together:end -->


## Descriptors are slots

A RISC OS socket descriptor is a small number, and programs build `fd_set` bitmaps
indexed by it. HostNet keeps a table of 256 slots, the same size as the Internet
module's socket table, which is also what `Socket_Gettsize` reports. Each slot holds
a host socket or −1, and the guest's descriptor *is* the slot number. New sockets
take the lowest free slot, as the Internet module does, because, as the comment on
`hn_alloc` puts it, "programs do notice if descriptors come back in a different
order".


<!-- doccrate:keep-together:start -->

### What each slot records

| Field | Set by | Used for |
|:---|:---|:---|
| `fds[]` | `Creat`, `Accept` | the host socket, or −1 |
| `nonblock[]` | `Ioctl FIONBIO` | whether a would-block is `EWOULDBLOCK` or `HN_RC_RETRY` |
| `async[]` | `Ioctl FIOASYNC` | whether wake-ups raise Internet Event 19 |
| `owner[]` | `Ioctl FIOSETOWN` | recorded and returned; not otherwise used |
| `woke[]` | the wake-up poll | the last event reason reported (chapter 6) |
| `connecting[]` | `Connect` | a non-blocking connection is in flight (chapter 4) |

<!-- doccrate:keep-together:end -->


<!-- doccrate:keep-together:start -->

## Socket addresses, three shapes

A RISC OS socket address comes in two layouts, and the host's is a third. The new
layout, used by the `_1` SWIs, is a length byte, a family byte, the port, the address
and eight zero bytes. The old layout has a 16-bit family word in place of the first
two bytes. macOS's own layout has a length byte and Windows' does not. Ports and
addresses are in network byte order in all of them, so they are copied as bytes.
`hn_sa_in` builds a host address from either guest layout:

```c
static bool hn_sa_in(uint64_t addr, uint32_t len, struct sockaddr_in *sin)
{
    uint8_t g[HN_SA_LEN];

    if (len < 8 || len > HN_SA_LEN) {
        return false;
    }
    memset(g, 0, sizeof(g));
    if (!vmch_guest_rw(addr, g, len, false)) {
        return false;
    }
    memset(sin, 0, sizeof(*sin));
    sin->sin_family = AF_INET;
    memcpy(&sin->sin_port, g + 2, 2);
    memcpy(&sin->sin_addr, g + 4, 4);
    return true;
}
```

<!-- doccrate:keep-together:end -->


Both guest layouts put the port at offset 2 and the address at offset 4, so reading
them needs no knowledge of which layout it is. Writing an address back does need to
know: `hn_sa_out` sets a length byte and a family byte for the new layout, and a
16-bit family word for the old one.


<!-- doccrate:keep-together:start -->

### A worked address

`listen.bas` binds port 9000 on every interface, in the new layout. The bytes it
writes, and what the host builds from them:

| Offset | Bytes in guest memory | Meaning | Host `sockaddr_in` |
|:---|:---|:---|:---|
| `+0` | 16 | length (new layout) | not copied |
| `+1` | 2 | family, `AF_INET` | set to `AF_INET` whatever it says |
| `+2` | 35, 40 | port 9000 in network order: 35 × 256 + 40 | `sin_port`, copied as two bytes |
| `+4` | 0, 0, 0, 0 | address 0.0.0.0, any interface | `sin_addr`, copied as four bytes |

<!-- doccrate:keep-together:end -->


<!-- doccrate:keep-together:start -->

## Message flags: translated, never passed through

The flag word of `send`, `recv` and the message calls is translated bit by bit, and
unknown bits are dropped. NetSurf's DNS library, c-ares, sets 0x400, its own
`MSG_NOSIGNAL`, on every send. The host rejected the unknown bit with `EOPNOTSUPP`,
and, in Sprint 2's words, "every HTTPS fetch failed at the first byte while the trace
showed a perfectly good connected socket":

```c
static int hn_msgflags(uint32_t g)
{
    int h = 0;

    if (g & 0x1) {
        h |= MSG_OOB;
    }
    if (g & 0x2) {
        h |= MSG_PEEK;
    }
    if (g & 0x4) {
        h |= MSG_DONTROUTE;
    }
#ifdef MSG_WAITALL
    if (g & 0x40) {
        h |= MSG_WAITALL;
    }
#endif
#ifdef MSG_NOSIGNAL
    h |= MSG_NOSIGNAL;
#endif
    return h;
}
```

<!-- doccrate:keep-together:end -->


`MSG_NOSIGNAL` is always added where the host has it, because a `SIGPIPE` from a
guest writing to a closed connection would kill the emulator, not the guest. For
macOS, which the source notes has no `MSG_NOSIGNAL`, `hn_no_sigpipe` sets
`SO_NOSIGPIPE` on every socket instead.


<!-- doccrate:keep-together:start -->

## Socket options, by name

The numbers agree on RISC OS, Windows and macOS, but options are still translated
by name. An unknown one is accepted and ignored: "a program that cannot set SO_DEBUG
should carry on, not fail".

| Level | Options served, by guest number | On the host |
|:---|:---|:---|
| `SOL_SOCKET`, 0xFFFF | `REUSEADDR` 0x4, `KEEPALIVE` 0x8, `BROADCAST` 0x20, `LINGER` 0x80, `OOBINLINE` 0x100, `SNDBUF` 0x1001, `RCVBUF` 0x1002, `ERROR` 0x1007, `TYPE` 0x1008 | the same names; `SO_ERROR` translated |
| `IPPROTO_TCP`, 6 | `TCP_NODELAY` 0x1 | `TCP_NODELAY` |
| anything else | any | set: ignored; get: `ENOPROTOOPT` |

<!-- doccrate:keep-together:end -->


Option values of up to 32 bytes are passed through otherwise unchanged.


<!-- doccrate:keep-together:start -->

## Ioctls

Five requests are served, by the guest's own numbers:

| Request | Number | What HostNet does |
|:---|:---|:---|
| `FIONBIO` | 0x8004667E | records the program's non-blocking choice |
| `FIONREAD` | 0x4004667F | writes 1 if the socket is readable, else 0 |
| `FIOASYNC` | 0x8004667D | records whether to raise Internet Event 19 for this socket |
| `FIOSETOWN`, `FIOGETOWN` | 0x8004667C, 0x4004667B | records and returns the owner |
| anything else | | `EOPNOTSUPP` |

<!-- doccrate:keep-together:end -->


`FIOASYNC` has a story. In Sprint 1 it was going to be refused until the events
existed, but the stock Resolver module sets it immediately after creating its socket,
and closes the socket if the request fails. The comment in `hn_ioctl` records the
result of refusing it: it "is why DNS did nothing at all the first time Sprint 1 was
tested".

## `Sysctl`, for one line of the boot

`Socket_Sysctl` reads and sets the network stack's tunable values. HostNet has none,
but it cannot refuse the call. `!Internet`'s start-up sets one value, the UDP
checksum switch, under `CheckError`, so a refusal would stop the boot before the file
that sets the name servers. HostNet accepts every `Sysctl`, and a request to read a
value gets a length of zero back.

## Error numbers, by name

The host reports failures through `errno`, whose numbers are the host's: native on
macOS, and on Windows the C runtime's, which QEMU's wrappers set from Winsock's
error. RISC OS programs expect 4.4BSD numbers. `hn_errno` therefore maps by *name*,
with a `case` for each error a socket call can produce, and returns RISC OS's
numbers:


<!-- doccrate:keep-together:start -->

#### The RISC OS numbers

| Group | Errors and their RISC OS numbers |
|:---|:---|
| general | `EINTR` 4, `EBADF` 9, `EACCES` 13, `EFAULT` 14, `EINVAL` 22, `EMFILE` 24 |
| would block | `EWOULDBLOCK` 35, also from `EAGAIN`; `EINPROGRESS` 36; `EALREADY` 37 |
| the socket and protocol | `ENOTSOCK` 38, `EDESTADDRREQ` 39, `EMSGSIZE` 40, `EPROTOTYPE` 41, `ENOPROTOOPT` 42, `EPROTONOSUPPORT` 43, `EOPNOTSUPP` 45, `EAFNOSUPPORT` 47 |
| addresses and networks | `EADDRINUSE` 48, `EADDRNOTAVAIL` 49, `ENETDOWN` 50, `ENETUNREACH` 51, `EHOSTUNREACH` 65 |
| connections | `ECONNABORTED` 53, `ECONNRESET` 54, `ENOBUFS` 55, `EISCONN` 56, `ENOTCONN` 57, `ETIMEDOUT` 60, `ECONNREFUSED` 61 |

<!-- doccrate:keep-together:end -->


Any host error not in the list becomes `EINVAL`. The module then builds the RISC OS
error block from the number: &20E00 plus the errno, with the error's name, such as
`CONNREFUSED`, as the message.


<!-- doccrate:keep-together:start -->

## Limits

| Limit | Value | Why |
|:---|:---|:---|
| sockets per machine | 256 | the Internet module's own table size |
| bytes per call | 1 MiB | "a bound on what the host will allocate" |
| `readv`/`writev` entries; option value | 1024; 32 bytes | bounds on a bad count or length |
| `fd_set`; events per poll | 32 bytes; 16 | one bit per slot; the reply list |
| address family | IPv4 only | `AF_INET6` answers `EAFNOSUPPORT` |

<!-- doccrate:keep-together:end -->


## Windows and POSIX

The first working build of the device was on Windows. `20bf267db5` made it build on
macOS, and the two hosts differ in a handful of places:


<!-- doccrate:keep-together:start -->

#### The two hosts

| Concern | Windows | macOS and other POSIX hosts |
|:---|:---|:---|
| **descriptor** | a C runtime descriptor wrapping a `SOCKET` | a native descriptor |
| **non-blocking** | `ioctlsocket`, wrapped by QEMU | `fcntl` with `O_NONBLOCK` |
| **readiness** | `WSAPoll` on `_get_osfhandle(fd)` | `poll`, with `<poll.h>` included explicitly for macOS |
| **closing** | Winsock's `closesocket` | `close`, through `#define closesocket close` |
| **SIGPIPE** | not applicable | `MSG_NOSIGNAL` where it exists, `SO_NOSIGPIPE` on macOS |

<!-- doccrate:keep-together:end -->

