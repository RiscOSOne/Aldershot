# 5. Conclusions

<!-- doccrate:keep-together:start -->
## What this says about WimpForth

WimpForth's save model assumes the flat machine of its era: the image at `&8000`,
everything between `&8000` and `here` already touched, and an application slot that fits
snugly. Under a modern, large WimpSlot, `here` — or the layout the kernel derives from
`OS_GetEnv`'s RAM limit — leaves the saved range spanning pages that have never been
touched, and `OS_File 10`'s buffered SVC-mode copy is not prepared for that.

This is a genuine portability defect in WimpForth, not a misconfigured farm. The same
binary works in a 2 MB slot and dies in a big one.
<!-- doccrate:keep-together:end -->

<!-- doccrate:keep-together:start -->
## Why kernel builds always worked

The metacompiler is unaffected. `TARGET-SAVE` saves a buffer it has only just written,
so every page of it is mapped. That is why rebuilding the kernel always worked while
rebuilding the application died.
<!-- doccrate:keep-together:end -->

<!-- doccrate:keep-together:start -->
## The bootstrap now completes

With two small fixes in the WimpForth tree — touching the range before `save-file`, and
restoring the Latin-1 delimiters — the full documented bootstrap completes on the farm.

| Stage | Result |
|:---|:---|
| Metacompile `kernel32` | byte-identical to upstream |
| `fload makewin` on the bare kernel | loads |
| Save `FWIN` | 159,984 bytes written |
| Run the new `FWIN` | boots and computes: `1 2 + .` gives `3 ok` |
<!-- doccrate:keep-together:end -->

The freshly built `FWIN32` is committed to the fork.

<!-- doccrate:keep-together:start -->
## What is not yet established

These are why the case stays open.

| Question | What would settle it |
|:---|:---|
| Which kernel instruction is at `UtilityModule` `+&17FB4`, and whose copy loop it is | a ROM with symbols, or `*Where` before the handler runs |
| Whether the box's `&20006E08` is the real faulting PC, or an artefact of the handler | the RAMFS error box, captured without a primed prompt |
| Why the saved range reaches past mapped memory in a big slot | a read of WimpForth's RAM-limit layout |
<!-- doccrate:keep-together:end -->

On the last of these: WimpForth's cold start puts `rp` at `OS_GetEnv`'s RAM limit less
16 KB, and the end of the dictionary reportedly lands at the top of a 32 MB slot. Whether
that is what carries the save range past the mapped pages is exactly what has not yet
been read.
