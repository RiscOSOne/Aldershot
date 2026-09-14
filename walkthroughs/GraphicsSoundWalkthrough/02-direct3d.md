# 2. The window owns the thread

The fork was first built on Windows. QEMU's own display back ends convert the
guest's framebuffer on the CPU and hand it to GTK or SDL. The fork replaced that
with a window it owns outright: `-display dx11`. This chapter walks through that
front end — its threads, the boundary between C and C++, the swap chain that
cannot wedge, the per-frame upload and decode, and the input path back to the
guest.


<!-- doccrate:keep-together:start -->

## Threads, and who may touch what

QEMU is a program, not a library, but it has a documented way for a display back
end to take the main thread. It was added for the Cocoa port. `system/main.c`
runs `qemu_init()` on the main thread; if the back end has set the `qemu_main`
function pointer, QEMU moves its main loop to a new thread and calls that instead.
The Windows front end uses exactly that hand-off:

| Thread | Owns | Touches QEMU state |
|:---|:---|:---|
| UI (the main thread) | the window, Direct3D 11, input state | reads guest RAM and the framebuffer config without locks; takes QEMU's big lock only to inject input |
| `qemu_main` | QEMU's main loop | everything, as usual |
| four vCPU threads | the emulated Cortex-A72s | everything, as usual |

<!-- doccrate:keep-together:end -->


The design record states the rule for the frame loop plainly: the UI **never holds
the big lock while rendering**. A frame reads a few words of configuration, reads
the framebuffer bytes, uploads them and draws. The guest writes its screen under no
lock either, and the design record accepts the consequence: a torn frame is exactly
what a real monitor shows mid-update. Chapter 7 is about what happened when that
acceptance was tested.


<!-- doccrate:keep-together:start -->

## A C boundary for a C++ window

Direct3D is miserable from C and ordinary from C++. QEMU's headers do not parse as
C++. So the front end is split in two at an `extern "C"` boundary:

| File | Language | Holds |
|:---|:---|:---|
| `ui/dx11.c` | C | QEMU glue: the framebuffer view, input injection, options |
| `ui/dx11.cpp` | C++17 | the Win32 window, Direct3D device, shaders and frame loop |
| `ui/dx11.h` | both | the boundary: plain structs and functions, no QEMU types |

<!-- doccrate:keep-together:end -->


The heart of the boundary is one struct. The glue fills it once per frame on the UI
thread, and the C++ side draws from it:

```c
typedef struct Dx11FbView {
    uint32_t xres, yres;            /* visible size, pixels */
    uint32_t xoffset, yoffset;      /* pan within the buffer */
    uint32_t pitch;                 /* bytes per buffer row */
    uint32_t bpp;                   /* guest pixel format, bits */
    uint32_t pixo;                  /* 1 = RGB order, 0 = BGR */
    uint32_t rows;                  /* mapped buffer rows (>= yres+yoffset) */
    uint32_t generation;            /* config generation of this view */
    uint64_t gbase;                 /* guest address of the buffer */
    const void *fb;                 /* raw framebuffer bytes, pitch * rows */
    const void *palette;            /* 256 * 4 bytes, 0x00BBGGRR; always set */
} Dx11FbView;
```

**The pointers are host addresses into guest RAM.** There is no copy at this
boundary.

## Mapping the screen once per mode

The glue maps the framebuffer and palette with `address_space_map` and keeps the
mapping until the configuration's generation moves:

```c
gen = bcm2835_fb_get_config(view.fb, &cfg);
if (!cfg.xres || !cfg.yres) {
    return 0;                   /* no mode programmed yet */
}
if (view.mapped && gen != view.generation) {
    address_space_unmap(&view.fb->dma_as, view.fb_ptr, view.fb_len, false, 0);
    address_space_unmap(&view.fb->dma_as, view.pal_ptr, 256 * 4, false, 0);
    view.mapped = false;
}
if (!view.mapped) {
    ...
    view.fb_ptr = address_space_map(&view.fb->dma_as, cfg.base, &fb_len,
                                    false, MEMTXATTRS_UNSPECIFIED);
    view.pal_ptr = address_space_map(&view.fb->dma_as,
                                     view.fb->vcram_base, &pal_len,
                                     false, MEMTXATTRS_UNSPECIFIED);
    ...
}
```

Two details matter. The mapping covers exactly what the guest indexes: the visible
rows plus any pan offset. And pan offsets count only when the virtual size exceeds
the visible one — the same rule the framebuffer model applies, mirrored here so the
image does not shift.

## A swap chain that cannot wedge

The first version presented with a plain blocking `Present(1)`, and it hung the
whole UI in a way worth recording. A flip-model present blocks until the Desktop
Window Manager (DWM) retires a buffer. If the window stops being composed for five
seconds — occluded, minimised, or a DWM hiccup — Windows marks it *Not Responding*,
DWM stops flipping it, and the one thread that pumps messages is stuck forever. A
debugger found the thread parked inside DXGI.

Commit `a59d153d17` makes the chain **waitable**:

```c
scd.SwapEffect        = DXGI_SWAP_EFFECT_FLIP_DISCARD;
scd.Flags             = DXGI_SWAP_CHAIN_FLAG_FRAME_LATENCY_WAITABLE_OBJECT;
...
HRESULT hrl = swap2->SetMaximumFrameLatency(1);
dx11.swap_wait = swap2->GetFrameLatencyWaitableObject();
```

The frame loop then waits for a free latency slot *with the message queue armed*,
and presents only when a slot is actually free:

```c
if (dx11.swap_wait) {
    DWORD w9 = MsgWaitForMultipleObjectsEx(
        1, &dx11.swap_wait, 100, QS_ALLINPUT, MWMO_INPUTAVAILABLE);
    if (w9 != WAIT_OBJECT_0) {
        continue;           /* message or timeout: no free slot */
    }
} else {
    Sleep(1);
}
if (dx11_acquire_target()) {
    if (!dx11_render_frame()) {
        dx11.context->ClearRenderTargetView(dx11.rtv, clear);
    }
    HRESULT hr = dx11.swap->Present(1 /* vsync */, 0);
    ...
}
```

Input keeps flowing while the display is stalled. A hardware device is tried
first, then WARP, Direct3D's software renderer. Drivers without the waitable
interface fall back to the older blit-model chain: its present copies rather than
flips, so it cannot wedge, and it paces on a sleep. A minimised window presents
nothing.

## Upload: two buffers, not one

Each frame copies the guest's bytes into a GPU buffer. The front end keeps **two**
copies, and a comment in the source explains why:

> A copy the guest wrote into while we were reading it splices two guest states
> together, and during a window move that puts part of the window where it used
> to be.

So the fresh copy goes into the buffer that is *not* on screen, and becomes the
shown one only if it came out whole. Chapter 7 covers how "whole" is decided.

Each buffer is a Direct3D `ByteAddressBuffer` of the whole framebuffer, filled with
one `Map(WRITE_DISCARD)` and one `memcpy`. It was a texture at first, until wide
modes hit the texture ceiling. A `Texture2D` is limited to 16,384 texels across. A
3840-wide 32 bpp mode has a 15,360-byte pitch, just inside the limit, and anything
wider overflowed. A flat byte buffer has no ceiling, and the pitch can be anything.

## Decode: one shader source, seven decoders

The raw bytes are turned into colour on the GPU, at the guest's own resolution. One
HLSL source is compiled at start-up into seven pixel shaders, for 32, 24, 16, 8, 4,
2 and 1 bits per pixel, selected by a `BPP` macro. The 32 and 16 bpp cases show the
shape:

```hlsl
uint fb_word(uint row, uint byte_off)
{
    return raw.Load(row * P.dim.z + (byte_off & ~3u));
}

float4 ps_main(VSOut v) : SV_Target
{
    uint x = (uint)v.pos.x;
    uint y = (uint)v.pos.y;

    /* visible pixel -> buffer coordinates, pan included */
    uint bx = x + P.misc.x;
    uint by = y + P.misc.y;

    uint r = 0, g = 0, b = 0;
#if BPP == 32
    uint w = fb_word(by, bx * 4);          /* aligned by construction */
    r = w & 0xff;
    g = (w >> 8) & 0xff;
    b = (w >> 16) & 0xff;
    if (P.misc.z == 0) { uint t = r; r = b; b = t; }   /* BGR order */
#elif BPP == 16
    uint w = fb_word(by, bx * 2);          /* two pixels per word */
    uint sh = (bx & 1u) * 16;              /* odd pixel is the high half */
    w >>= sh;
    r = ((w >> 11) & 31) * 255 / 31;
    g = ((w >>  5) & 63) * 255 / 63;
    b = ((w      ) & 31) * 255 / 31;
#else
    /* palettised: 8 bpp, or sub-byte 1/2/4 with LSB-first packing */
    uint bpp = P.dim.w;
    uint byte = fb_byte(by, bx * bpp / 8);
    uint idx = (byte >> ((bx * bpp) & 7)) & ((1u << bpp) - 1);
    float4 c = pal.Load(int3(idx, 0, 0));
    return float4(c.rgb, 1);
#endif
    return float4(r / 255.0, g / 255.0, b / 255.0, 1);
}
```

A byte buffer loads whole words, so each depth extracts its bytes from aligned
words. The palette is a 256 × 1 texture, refilled only when its bytes change. In
practice the video driver only ever asks for 8 and 32 bpp. The other decoders come
free, and all seven were checked pixel-exact on synthetic buffers.

## Scale: the window is the monitor

A second pass stretches the decoded image over the whole client area. The design
record puts it this way: the window is the monitor, so every mode fills it, as a
Pi's GPU fills a real display. Three scalers are offered with
`-display dx11,scaling=sharp|linear|nearest`, plus optional scanlines:

```hlsl
#if SCALING == 1
    /* Sharp bilinear: within each source texel the bilinear transition
     * is narrowed to a 1/ratio-wide band at the texel edge, so at 1:1
     * the image passes through untouched and at 2x+ it is crisp with
     * just enough filtering to avoid hard staircases. */
    float2 ratio = max(outPx / srcPx, 1.0);
    float2 halfw = 0.5 - 0.5 / ratio;         /* half the band width */
    float2 i = floor(st);
    float2 f = st - i - 0.5;                  /* -0.5..0.5 in-texel */
    st = i + 0.5 + clamp(f, -halfw, halfw);
#elif SCALING == 2
    st = floor(st) + 0.5;                     /* nearest */
#endif
```

One Direct3D contract cost a black screen. The decoded surface has a full mip chain
for downscaling, but it had been created without the `GENERATE_MIPS` flag.
`GenerateMips` is silently a no-op without it. The moment the window was smaller
than the guest mode, the sampler picked a mip level that had never been written,
and the frame went black (commit `21db90950e`).

## Input: scan codes and an absolute pointer

**Keys.** Windows delivers a scan code in each key message, and QEMU's AT set 1
keymap is indexed by exactly that. The first version indexed QEMU's *win32* keymap
instead, which is keyed by virtual-key codes. Scan code `0x2c` is `z`, but virtual
key `0x2c` is Print Screen. So typing `z` opened NetSurf's print dialogue, and later
letters rewrote the URL bar. That was the "desktop freezes after a few keys" of the
early bring-up (commit `c84d1bbc7d`).

The fix keys the lookup on the scan code and its `0xE0` prefix, and injects under
the big lock:

```c
uint32_t scancode = (lparam >> 16) & 0xff;
uint32_t code = (lparam & (1u << 24)) ? (0xe000 | scancode) : scancode;
...
unsigned int lnx = qemu_input_map_atset1_to_linux[code];
...
bql_lock();
qemu_input_event_send_key_linux(NULL, lnx, down);
bql_unlock();
```

**The pointer** is absolute. Ungrabbed, the host cursor's position in the client
area is scaled onto the guest screen, so the two arrows stay together. Grabbed,
host deltas accumulate into a position that is still sent absolutely. Motion feels
relative, but it cannot drift, and releasing the grab realigns the arrows.

**Three buttons, all for the guest.** RISC OS is built around Select, Menu and
Adjust, and Menu is the *middle* button, the one the desktop uses most. A front end
that kept the middle button for itself would leave the guest unable to open a menu.
So all three buttons go to the guest, and the grab moved to `Ctrl+Alt+G`. Losing
focus always releases it.


<!-- doccrate:keep-together:start -->

### What else the window does

| Feature | How |
|:---|:---|
| full screen | `Alt+Enter`, borderless, never sent to the guest |
| DPI | per-monitor aware; the default window is 800×600 at the system scale |
| auto-resize | once, to the guest's settled mode, after about 3 s; then the size is the user's |
| screenshots | Print Screen writes the decoded surface to a PNG |
| snapshots | *Load snapshot* in the system menu, run as a bottom half on QEMU's loop |
| mode changes | a new generation rebuilds the per-mode pipeline without moving the window |

<!-- doccrate:keep-together:end -->


The acceptance test compared the window's decode with QEMU's own screendump of the
guest framebuffer. **84 of 480,000 pixels differed**, and all of them were the clock,
which had ticked between the two captures.
