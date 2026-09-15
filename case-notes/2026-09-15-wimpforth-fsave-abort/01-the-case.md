# 1. The case as filed

<!-- doccrate:keep-together:start -->
## What works

WimpForth runs as `FWIN32`, an Absolute file of 159,904 bytes, launched by `WimpTask`
from `HostFS:$.WimpForth`. The upstream is [alban-read/forth](https://github.com/alban-read/forth); the case was
worked on a local fork at `<redacted>`.

| Test | Result |
|:---|:---|
| The interpreter: `1 2 + .` | `3 ok` |
| Loading source with `FLOAD` | works |
| Full metacompilation: `FLOAD META32` | a 27,020-byte kernel image |
| `TARGET-SAVE kernel32` | written to HostFS, byte-identical to the shipped `kernel32`, every time |
<!-- doccrate:keep-together:end -->

The saved kernel's MD5 is `aefefe1dcf1fa8c123f01ad5f4de4266`, the same as upstream's.

<!-- doccrate:keep-together:start -->
## What dies

Building the application is a different story. `FLOAD MAKEWIN`, or simply
`fsave TESTSAVE`, compiles around twenty source files and prints
`Extensions Loaded, 3054 words in dictionary`. Then comes the `FSAVE` step, which dumps the
whole running image to a file through `OS_File 10`, and the machine dies:

```
Application has gone wrong: abort on data transfer at &2006E08
```
<!-- doccrate:keep-together:end -->

In a TaskWindow the same step does not raise an error box: it simply wedges, with no
`ok` prompt. After the error box the desktop is unusable, and the virtual machine has to
be restarted.

<!-- doccrate:keep-together:start -->
## The save path

The save goes through two short pieces of WimpForth, both upstream and unmodified.

#### The kernel's `OS_File` word, and `save-file`

```forth
code OS_File                    \ r0 <- TOS, then r1..r5 from stack
  mov r0, tos
  ldmfd sp !, { r1, r2, r3, r4, r5, tos }
  swi x " OS_File"
next c;
: save-file ( ad len filename -- )
  1+ >r bounds 0 &ff8 r> 10 OS_File ;
```
<!-- doccrate:keep-together:end -->

<!-- doccrate:keep-together:start -->
#### `"fsave` in UTILS

```forth
: "fsave ( a1 n1 -- )
    fsave-buf place fsave-buf count over >r + 0 swap c!
    32768 here over - r> 1-      \ start=&8000, end=here
    save-file ;
```
<!-- doccrate:keep-together:end -->

`"fsave` saves everything from `32768` — `&8000`, the classic base of application space —
up to `here`, the top of the dictionary, as an Absolute file (type `&FF8`).

## The first theory

The theory filed with the case was that the hardcoded `32768` image base was wrong for a
task running higher in memory. Chapter 2 is where that was re-examined.
