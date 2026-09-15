# 3. Painting the tag from the guest

RISC OS has to put the tag on exactly the pixels that are background, keep it
there while windows move, and never put it anywhere else. This chapter shows how
the disc does that with one generated sprite and two edited lines. It walks
through the tool that writes the sprite, byte by byte, and explains why the tile is
256 pixels. It ends with the release scripts that make the change to the disc a
user installs.

## No RISC OS code changes

Three facts about RISC OS make the approach possible. MACOS.md §7a records each of
them as measured on a running machine, by reading the framebuffer out of guest
memory:

- **Pinboard already tiles sprites.** `*Backdrop -Tile` plots a sprite repeatedly
  across the whole background. The tag only has to be in the sprite.
- **The kernel's sprite plot writes all 32 bits.** A sprite pixel's transfer byte
  reaches the framebuffer unchanged. The tag therefore lands on exactly the
  background, and windows, menus, icons and their labels, which RISC OS draws
  itself, stay untagged.
- **The Wimp's block copies carry the byte with the pixels.** When a window is
  dragged, the Wimp copies screen blocks and asks Pinboard to redraw the area it
  uncovers. Pinboard's redraw re-tags that area, and nothing stale is left behind.

Without the layer, the tile is simply the sage ground. The old watermark sat on the
same colour, so the disc looks like a plain Acorn desktop on any other display.


<!-- doccrate:keep-together:start -->

## Two edited lines

The Acorn theme's boot files change in two places:

| File | Before | After |
|:---|:---|:---|
| `!Boot/Choices/Boot/Tasks/PinSetup` | `Backdrop -Centre …Themes.Acorn.Backdrop` | `Backdrop -Tile …Themes.Acorn.BackTile` |
| `!Boot/Choices/Boot/PreDesk/ThemeSetup` | `WimpVisualFlags -RemoveIconBoxes` | `WimpVisualFlags -RemoveIconBoxes -NoIconBoxesInTransWindows` |

<!-- doccrate:keep-together:end -->


The first line swaps the centred watermark sprite for the tiled, tagged one. The
`…` stands for `Boot:Resources.!ThemeDefs`, and the rest of the line, including
the sage `-Colour`, is unchanged.

The second line is less obvious. By default the Wimp fills a small box behind the
name of each icon on the pinboard. That box is drawn by the Wimp, so it is
untagged. Over the host's layer
it would appear as a solid rectangle behind every icon name on the desktop. The
flag `-NoIconBoxesInTransWindows` tells the Wimp not to fill those boxes, so the
names are drawn straight onto the tagged tile.


<!-- doccrate:keep-together:start -->

## The tool: `mkbacktile.py`

`riscos-pi4/tools/mkbacktile.py` writes the sprite file. It is 68 lines, with no
dependencies beyond Python's `struct` module. The core is one function:

```python
KEY = (0xB7, 0xC0, 0xB4)      # R, G, B: the Acorn theme's sage
TAG_BELOW = 0x80              # transfer byte: layer tag 10, reserved bits 0


def tile(size):
    r, g, b = KEY
    word = struct.pack('<I', r | (g << 8) | (b << 16) | (TAG_BELOW << 24))
    body = word * (size * size)
    hdr = bytearray(44)
    struct.pack_into('<i', hdr, 0, 44 + len(body))          # to the next sprite
    struct.pack_into('<12s', hdr, 4, b'backtile')
    struct.pack_into('<7i', hdr, 16,
                     size - 1,                              # width in words - 1
                     size - 1,                              # height - 1
                     0, 31,                                 # first and last bit used
                     44, 44,                                # image; mask = image: none
                     1 | (90 << 1) | (90 << 14) | (6 << 27))  # 32bpp, 90 dpi
    sprite = bytes(hdr) + body
    return struct.pack('<III', 1, 16, 16 + len(sprite)) + sprite
```

<!-- doccrate:keep-together:end -->


The pixel word is the `0x80B4C0B7` from chapter 2, repeated `size × size` times.
Everything else is the standard RISC OS sprite file layout, described below.


<!-- doccrate:keep-together:start -->

### The file header

A sprite file is a sprite area without its first word, the area's total size. So
the file starts with three words:

| Offset | Value | Meaning |
|:---|:---|:---|
| `+0` | 1 | the number of sprites in the file |
| `+4` | 16 | where the first sprite starts, counted as if the missing size word were there |
| `+8` | `16 + len(sprite)` | the first free word after the last sprite, counted the same way |

<!-- doccrate:keep-together:end -->


<!-- doccrate:keep-together:start -->

### The sprite header

The sprite itself starts with a 44-byte header, and the pixels follow at once:

| Offset | Value written | Meaning |
|:---|:---|:---|
| `+0` | `44 + len(body)` | the offset to the next sprite, which is the size of this one |
| `+4` | `backtile` | the name, 12 bytes, zero-padded |
| `+16` | `size − 1` | the width in words, minus one; at 32bpp a pixel is a word |
| `+20` | `size − 1` | the height in rows, minus one |

<!-- doccrate:keep-together:end -->


<!-- doccrate:keep-together:start -->

#### The sprite header, continued

| Offset | Value written | Meaning |
|:---|:---|:---|
| `+24`, `+28` | 0, 31 | the first and last bits used in each row, so every bit is used |
| `+32` | 44 | where the image starts, from the start of the sprite |
| `+36` | 44 | where the mask starts; equal to the image offset means **no mask** |
| `+40` | the mode word | the sprite type and resolution, below |

<!-- doccrate:keep-together:end -->


The tile has no mask, and that matters. Any pixel a mask marked transparent would
not be plotted, so it would keep whatever the framebuffer held before, untagged.


<!-- doccrate:keep-together:start -->

### The mode word

The last header word describes the pixel format instead of naming a numbered
screen mode:

| Bits | Field | Value |
|:---|:---|:---|
| 0 | set, so the word describes a format rather than a numbered mode | 1 |
| 1–13 | horizontal resolution, in dots per inch | 90 |
| 14–26 | vertical resolution, in dots per inch | 90 |
| 27–31 | the sprite type | 6: 32 bits per pixel |

<!-- doccrate:keep-together:end -->


Ninety dots per inch in both directions matches the square-pixel desktop modes the
emulator uses, so the tile is plotted pixel for pixel rather than scaled.

## Why 256 pixels

A 256×256 tile makes a file of 12 + 44 + 256 × 256 × 4 = 262,200 bytes. That looks
wasteful for one colour, and the first version of the tool used a 32-pixel tile.
Commit `e80479db19` changed the default after measuring both.

Pinboard's tiled plot makes one sprite plot per tile. By simple arithmetic, a
256-pixel tile covers the 800×534 background in about a dozen plots, where a
32-pixel tile needs more than four hundred. GVFill carries each plot to the host,
so the difference is small, but it is measurable:


<!-- doccrate:keep-together:start -->

#### One full background redraw, measured

M4 Max, end-user disc, GVFill on:

| Tile | Per full redraw of 800×534 |
|:---|:---|
| 256 pixels (the default) | 0.156 ms |
| 32 pixels | 0.170 ms |

<!-- doccrate:keep-together:end -->


The size of the file does not matter in practice. It is 262 KB of a single
repeated word, which deflates to 325 bytes, so the release's disc zip barely grows.
Booted through Pinboard, both sizes tagged the same 424,465 background pixels, with
none left untagged.


<!-- doccrate:keep-together:start -->

## The release does it for you

Both release scripts apply the same change to the disc they ship. The Mac's
`make-release.sh` does it with a short Python script inside the shell script:

```python
def patch(rel, old, new, done):
    path = f"{disc}/{rel}"
    s = open(path, "rb").read().decode("latin-1")
    if done in s:
        return
    if s.count(old) != 1:
        sys.exit(f"{rel}: expected one '{old}'")
    open(path, "wb").write(s.replace(old, new).encode("latin-1"))
patch("!Boot/Choices/Boot/Tasks/PinSetup,feb",
      "Backdrop -Centre Boot:Resources.!ThemeDefs.Themes.Acorn.Backdrop",
      "Backdrop -Tile Boot:Resources.!ThemeDefs.Themes.Acorn.BackTile",
      "Themes.Acorn.BackTile")
patch("!Boot/Choices/Boot/PreDesk/ThemeSetup,feb",
      "WimpVisualFlags -RemoveIconBoxes",
      "WimpVisualFlags -RemoveIconBoxes -NoIconBoxesInTransWindows",
      "-NoIconBoxesInTransWindows")
```

<!-- doccrate:keep-together:end -->


Before the patch runs, the shell script checks that the disc has an Acorn theme
folder, and writes `BackTile,ff9` into it with `mkbacktile.py`. The `,feb` suffix
marks an Obey file and `,ff9` a sprite file, as HostFS names them on the host.


<!-- doccrate:keep-together:start -->

### Three guards in eight lines

The `patch` function is small, but it refuses to damage a disc it does not
recognise:

| Guard | What it prevents |
|:---|:---|
| return if the `done` text is already there | patching the same disc twice |
| exit unless `old` occurs exactly once | editing a disc whose boot files are not the expected shape |
| read and write as Latin-1 bytes | changing any byte outside the edit, whatever the file's character set |

<!-- doccrate:keep-together:end -->


On Windows, `make-release.py` has the same function under
`configure_backdrop(disc, scene)`, added in `d535505fad`. It uses `io.open` with
`newline=""`, so that Windows line-ending translation cannot alter the file, and
raises `SystemExit`, with its own wording, in the same two cases.


<!-- doccrate:keep-together:start -->

### What `BACKDROP` chooses

Both scripts take the scene as a setting: `BACKDROP=` for the shell script and
`--backdrop` for the Python one. It must be one of three values:

| Value | The disc | The app |
|:---|:---|:---|
| `acorn` (the default) | tile and Wimp flag added | starts with the acorn scene and the Backdrop menu |
| `acorn-live` | tile and Wimp flag added | starts with the moving acorn and the menu |
| `off` | unchanged: the centred watermark stays | the feature and its menu are left out |

<!-- doccrate:keep-together:end -->


The two scripts hand the value to the app differently. The Mac script stores it in
the app's `Info.plist` as `RISCOSBackdrop`, and the launcher reads it at each start.
The Windows script compiles it into the launcher executable with
`-DBACKDROP=L"acorn"`. Chapter 7 follows both routes. `d535505fad` shipped the
Windows version as the `RISCOSQEA72v7` release.
