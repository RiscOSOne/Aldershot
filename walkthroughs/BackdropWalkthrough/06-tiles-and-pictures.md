# 6. Tiles and pictures

Besides the two acorn scenes, the layer can show an image file in either of two
ways: repeated as a tile, or scaled to fill the window as a picture. This chapter
covers the value that names a scene, how each host loads an image and what it does
with transparency and colour, how big a tile is drawn on each host, how a picture
is fitted, and what a picture costs compared with drawing it inside RISC OS.


<!-- doccrate:keep-together:start -->

## Naming a scene

The same string names a scene everywhere: in `-display metal,backdrop=` or
`-display dx11,backdrop=`, in the Mac's saved `backdrop` setting, and in the value
each Backdrop menu item stores:

| Value | Scene |
|:---|:---|
| `off` | the feature gated out, which is only possible at start-up (chapter 7) |
| `none` | the feature on, nothing drawn: the guest's own sage shows |
| `acorn`, `acorn-live` | the distance-field acorn, still or moving (chapter 5) |

<!-- doccrate:keep-together:end -->


<!-- doccrate:keep-together:start -->

#### Naming a scene, continued

| Value | Scene |
|:---|:---|
| `tile:<file>` | the image repeated from the top left of the window |
| `picture:<file>` | the image scaled to fill the window, centred, with the overhang cropped |
| `<file>` | a bare path is a picture |

<!-- doccrate:keep-together:end -->


<!-- doccrate:keep-together:start -->

#### Parsing the value

Both front ends parse the value in a function called `backdrop_set_scene`. This is
the Windows one:

```c
    if (!spec || !*spec || !strcmp(spec, "none") || !strcmp(spec, "off")) {
        mode = DX11_BACKDROP_OFF;
        spec = "none";
    } else if (!strcmp(spec, "acorn")) {
        mode = DX11_BACKDROP_ACORN;
    } else if (!strcmp(spec, "acorn-live")) {
        mode = DX11_BACKDROP_ACORN_LIVE;
    } else if (!strncmp(spec, "tile:", 5)) {
        mode = DX11_BACKDROP_TILE;
        path = spec + 5;
    } else if (!strncmp(spec, "picture:", 8)) {
        mode = DX11_BACKDROP_PICTURE;
        path = spec + 8;
    } else {
        mode = DX11_BACKDROP_PICTURE;
        path = spec;
    }
    if (path) {
        if (backdrop_load_image(path)) {
            if (mode == DX11_BACKDROP_TILE && strstr(path, "@2x")) {
                backdrop.image_scale = 2.0;
            }
        } else {
            mode = DX11_BACKDROP_ACORN;
            spec = "acorn";
            dx11_log("backdrop: cannot read %s; using acorn", path);
        }
    }
```

<!-- doccrate:keep-together:end -->


Two behaviours are visible here. First, `off` reaching this function is treated as
`none`, because by the time a scene is being set the gate has already been decided.
Second, an image that cannot be read falls back to the acorn. The function's
comment gives the reason: "so the window is never a mystery black". A tile whose
path contains `@2x` is drawn at twice the density, which is covered below.

## Loading an image, two ways

Each front end loads images with its platform's own library and converts them to a
texture the scale pass can sample. The results are meant to match, but the details
differ.


<!-- doccrate:keep-together:start -->

### The two loaders compared

| | Mac: ImageIO and Core Graphics | Windows: WIC |
|:---|:---|:---|
| **reads** | anything ImageIO can decode, including HEIC | anything WIC has a decoder installed for |
| **transparency** | drawn over an opaque sage fill | converted to straight BGRA, then blended over sage in a loop |
| **texture** | `RGBA8Unorm`, no mipmaps | `B8G8R8A8_UNORM`, immutable, no mipmaps |

<!-- doccrate:keep-together:end -->


<!-- doccrate:keep-together:start -->

### Sage under the transparent parts

A PNG with transparent areas must not show black where it is transparent, because
the texture is later drawn as opaque colour. Both loaders put sage underneath. On
the Mac, Core Graphics does it by filling the bitmap context before drawing the
image into it:

```objc
            CGContextSetRGBFillColor(ctx, 0xB7 / 255.0, 0xC0 / 255.0,
                                     0xB4 / 255.0, 1.0);
            CGContextFillRect(ctx, CGRectMake(0, 0, w, h));
            CGContextSetInterpolationQuality(ctx, kCGInterpolationHigh);
            CGContextDrawImage(ctx, CGRectMake(0, 0, w, h), img);
```

<!-- doccrate:keep-together:end -->


The bitmap context is created with the sRGB colour space, so Core Graphics also
converts an image tagged with another colour space into sRGB as it draws.


<!-- doccrate:keep-together:start -->

#### The Windows blend

On Windows, WIC converts the image to straight (non-premultiplied) BGRA, and a loop
does the blend in integers:

```c
        /* Over the key colour, so transparency lands on sage. */
        for (size_t i = 0, n = (size_t)w * h; i < n; i++) {
            uint8_t *q = pix + i * 4;
            unsigned a = q[3];
            q[0] = (uint8_t)((q[0] * a + 0xB4 * (255 - a)) / 255);
            q[1] = (uint8_t)((q[1] * a + 0xC0 * (255 - a)) / 255);
            q[2] = (uint8_t)((q[2] * a + 0xB7 * (255 - a)) / 255);
            q[3] = 0xff;
        }
```

<!-- doccrate:keep-together:end -->


The byte order is blue, green, red, alpha, so blue is blended with `0xB4` and red
with `0xB7`. For example, a half-transparent pure red pixel (alpha 128) gets a red
channel of (255 × 128 + 183 × 127) / 255 = 219, a little lighter than full red. After
the loop every pixel is opaque.


<!-- doccrate:keep-together:start -->

## Tiles: an image pixel per point

A tile is sampled with the window pixel's own position, divided by the size of one
tile in window pixels:

```c
            float2 uv = v.pos.xy / (isz * float(P.bx.x) / 1000.0);
            bg = bgimg.Sample(rep, uv).rgb;
```

<!-- doccrate:keep-together:end -->


`isz` is the image size and `bx.x` is window pixels per image pixel, times 1000.
The `rep` sampler wraps, so the image repeats and each seam is filtered like any
other edge. The tile is anchored to the window's top left, not to the guest's
desktop, so RISC OS windows move over a pattern that stays still.

The two hosts compute `bx` differently. The Mac works in points, so a plain tile has
the same physical size on Retina and standard displays. Windows works in device
pixels, with no display scaling factor. The two lines come from different files, so
the comments naming them are added here:

```c
    p->bx[0] = (uint32_t)(px_per_point / backdrop.image_scale * 1000.0 + 0.5); /* Mac */
    p->bx[0] = (uint32_t)(1000.0 / backdrop.image_scale + 0.5);                /* Windows */
```


<!-- doccrate:keep-together:start -->

### How big a tile's pixel is drawn

| Host and file | Pixels per point | `image_scale` | `bx.x` | Window pixels per image pixel |
|:---|:---|:---|:---|:---|
| Mac, Retina, plain tile | 2 | 1 | 2000 | 2, which is one point |
| Mac, Retina, `@2x` tile | 2 | 2 | 1000 | 1, one device pixel |
| Mac, standard display, plain tile | 1 | 1 | 1000 | 1 |
| Mac, standard display, `@2x` tile | 1 | 2 | 500 | 0.5 |
| Windows, plain tile | — | 1 | 1000 | 1 |
| Windows, `@2x` tile | — | 2 | 500 | 0.5 |

<!-- doccrate:keep-together:end -->


<!-- doccrate:keep-together:start -->

## Pictures: fill and crop

A picture is scaled by the larger of the two ratios between window and image, so it
covers the window in both directions, and then centred:

```c
            float s = max(outPx.x / isz.x, outPx.y / isz.y);
            float2 uv = (v.pos.xy - 0.5 * outPx) / (s * isz) + 0.5;
            bg = bgimg.Sample(lin, uv).rgb;
```

<!-- doccrate:keep-together:end -->


In the other direction the picture overhangs the window and is cropped equally at both ends.


<!-- doccrate:keep-together:start -->

### A worked fit

The 6016×6016 HEIC picture from MACOS.md, on the Mac, in a 1646×1156 window:

| Step | Value |
|:---|:---|
| loaded size, after the 4096 cap | 4096 × 4096 |
| ratios, width and height; `s` is the larger | 1646 / 4096 = **0.402**, 1156 / 4096 = 0.282 |
| drawn size | 1646 × 1646 window pixels |
| cropped | 245 window pixels at the top and at the bottom |

<!-- doccrate:keep-together:end -->


<!-- doccrate:keep-together:start -->

## What a picture costs

The reason for drawing pictures on the host was cost. MACOS.md measured each way of
showing a picture behind the desktop, per full redraw of the 800×534 background by
RISC OS, on an M4 Max with GVFill on:

| How the picture is drawn | RISC OS, per full redraw | Host GPU, per frame at 1646×1156 |
|:---|:---|:---|
| inside RISC OS, as a JPEG | 48.3 ms | — |
| inside RISC OS, as a cached sprite | 0.505 ms | — |
| on the host, behind the tagged tile | 0.156 ms | 0.140 ms |

<!-- doccrate:keep-together:end -->


MACOS.md sums it up: against the plain Acorn look the layer is neutral, and against a
picture drawn inside RISC OS it is three times cheaper than a sprite and three
hundred times cheaper than a JPEG, whatever the host shows.
