# 9. Measured, and in progress

This chapter collects the numbers in one place: what the backdrop costs RISC OS, and
what each scene costs the GPU. It ends with the work on the backdrop that was in
progress, but not committed, on 15 September.

## What RISC OS pays

All measurements are from MACOS.md §7a, taken on an M4 Max with the end-user disc
and GVFill on. The first table is the cost to RISC OS of one redraw of the whole
800×534 background. Pinboard makes that redraw, in pieces, whenever a window moves
over the background:

<!-- doccrate:keep-together:start -->

#### One full background redraw

| Background | RISC OS, per full redraw |
|:---|:---|
| the old centred watermark, drawn by RISC OS | 0.118 ms |
| the tagged tile, 256 pixels (the default) | 0.156 ms |
| the tagged tile, 32 pixels | 0.170 ms |
| a picture as a cached sprite, drawn by RISC OS | 0.505 ms |
| a picture as a JPEG, drawn by RISC OS | 48.3 ms |

<!-- doccrate:keep-together:end -->

The tile costs 0.038 ms more than the old watermark per full redraw, which MACOS.md
calls neutral. What the host then draws through the tile, even a large picture,
costs RISC OS nothing more. Against a picture drawn inside RISC OS, the layer is
about 3 times cheaper than a cached sprite and 300 times cheaper than a JPEG.

<!-- doccrate:keep-together:start -->

## What the GPU pays

The second table is the scale pass's GPU time per frame, from an offscreen benchmark
of the Metal shader:

| Scene | 1646×1156 window | 3840×2160 window |
|:---|:---|:---|
| off | 0.034 ms | 0.054 ms |
| `acorn` | 0.037 ms | 0.095 ms |
| `acorn-live` | 0.040 ms | 0.136 ms |
| picture, a 4096×4096 image | 0.140 ms | 0.122 ms |
| tile | 0.019 ms | 0.058 ms |

<!-- doccrate:keep-together:end -->

The "off" row is the scale pass with no layer, so the other rows can be read against
it. The most expensive result, 0.14 ms, is less than 1% of a 60 Hz frame's 16.7 ms.
Before the acorn's early exit (chapter 5), the acorn cost 0.45 ms at 4K.

## Work in progress

The local checkout on 15 September held uncommitted changes to `ui/metal.m` and
MACOS.md, and a new, untracked `tools/mktiles.py`; this describes them as they stood.
The Mac's Backdrop menu was becoming a browser for large tile collections: a submenu
for each folder, filled only as it opens (`BackdropListMenu`), with a 32×16-point
swatch for every tile or picture and names without the `@2x` marker.

`mktiles.py` builds ready-made tile folders from four freely licensed sets, 196 tiles
in all: RISC OS's own desktop textures (Apache 2.0), the CDE desktop's backdrops (CC
BY-SA 3.0), the X.Org `xbitmaps` patterns (MIT-style) and Kenney's Pattern Pack Pixel
(CC0). Each source pixel becomes a 2×2 block in an `@2x` PNG, so it keeps its hard
edges, and each folder gets an "About these tiles.txt" with the source, the licence
and what was changed or left out.
