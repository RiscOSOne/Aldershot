# 4. A second bug on the way

<!-- doccrate:keep-together:start -->
## A rebuild that still failed

The first full rebuild with the workaround in place still failed, visibly and live on
the farm's display:

```
R stack underflow at DEBUG line 533
```
<!-- doccrate:keep-together:end -->

<!-- doccrate:keep-together:start -->
## One character, one byte

An earlier housekeeping change in the fork had transliterated the delimiter on the
PASM and DEBUG banner comments from `comment ö` to `comment oe`.

`comment` takes **one character** as the delimiter it skips to. With `oe` it took `o` —
and `o` occurs inside the commented block, in `#patchinto`. So the skip ended in the
middle of the block, and half of a dead definition executed.
<!-- doccrate:keep-together:end -->

<!-- doccrate:keep-together:start -->
## What the tree learned

Sources that are loaded into the guest are kept in Latin-1 precisely because constructs
like this depend on single bytes. Strings shown at run time may be plain ASCII, but a
delimiter has to keep its original byte.

The fix restored the Latin-1 `ö` delimiters (fork commit `<redacted>`).
<!-- doccrate:keep-together:end -->
