# plutovg

2D vector graphics for sysl, bound to [PlutoVG](https://github.com/sammycage/plutovg) — paths,
filling, stroking, dashing, gradients, clipping, transforms and text, rasterized into memory.

```
dependencies {
  plutovg { git = "github.com/sysl-lang/plutovg", version = "0.1.0" }
}
```

```sysl
import sh.sysl.plutovg

var s = surface(200, 200)
var c = canvas(&s)

c.rgb(0.1, 0.2, 0.9)
c.circle(100.0, 100.0, 60.0)
c.fill()

s.write_png("circle.png")
c.close()
s.close()
```

## It draws into memory and nothing else

What happens to those pixels afterwards — a PNG, a window, a 320×480 SPI panel — is somebody else's
problem. That is what makes this package independent of any display and of any microcontroller, and
testable on a desktop with no hardware at all.

A surface holds **premultiplied ARGB32**, which on a little-endian machine is the bytes B, G, R, A.
`data()` hands the whole buffer back.

## Why PlutoVG

The header was read rather than the README trusted, and the numbers are unusually good for a binding:

- **Not one public function takes or returns a struct by value.** Every parameter is a `float`, an
  `int`, a pointer or a `const T*`. **So this package has no `shim.c` at all** — the entire library is
  reachable by declaring it. The `UsefulBufC` problem that reshaped qcbor's whole API does not arise.
- **`float` throughout, not `double`** — which matters on a Cortex-M33, whose FPU is single-precision
  and which would otherwise emulate every coordinate in software.
- **`plutovg_surface_create_for_data`** — the caller owns the pixels. The house pattern, available
  straight off.
- **C99, no external dependencies**, 9 `.c` and 12 `.h` files, all vendored here.
- It carries its own TrueType rasterizer, so **text needs no second binding**.

## Rendering for a small machine

`for_data` borrows storage the caller already owns rather than allocating it:

```sysl
var band: [80 * 1024]u8
var s = for_data(band[..], 320, 64, 320 * 4)
```

A full 320×480 surface at 32bpp is **600 KiB**, more than an RP2350 has. A 320×64 band is 80 KiB,
which fits — so a program renders a band, pushes it, and reuses the buffer.

`sh.sysl.st7796` is the other half of that arrangement: it takes an ARGB32 band and converts to RGB565
as it pushes. **The conversion lives there rather than here**, so this stays a faithful binding, and
that is the only place the two packages touch.

## Closing things

`Surface`, `Canvas` and `Font` each hold a C allocation and each has a `close`. They are not released
for you — sysl has no way to adopt a raw pointer as owned storage, so the library's `destroy` is a call
you make. Closing twice is harmless: the handle is cleared, so the second call does nothing.

Storage lent by `for_data` is yours, and closing the surface does not free it.

## Capabilities

`alloc` and `posix`, and both are narrower than they look.

**`alloc`** is honest and not yet avoidable: PlutoVG mallocs its canvases, path storage and glyph
caches, and unlike QOI it has no `PLUTOVG_MALLOC` hook to redirect. `for_data` keeps the *pixels* out
of the heap, which is the large allocation and the reason band rendering is practical, but the canvas
itself still allocates.

**`posix` is `write_png` and `font`, and nothing else.** The whole drawing surface touches no file
system. A program that renders into its own buffer, loads a font from bytes linked into the binary with
`font_from_data`, and never writes a PNG uses no operating system at all — the capability is declared
because those two functions are part of the module, and a capability is a property of the module rather
than of the call.

## Tests

```
sysl test .
```

24 of them, and **they are pixel tests**. A binding can be wrong in a way that compiles, links and
returns without complaint — a parameter in the wrong order, a float read as a double, an enum off by
one — and the only thing that catches it is looking at what was drawn. So most of these rasterize
something small and read the bytes back: that red is red and not blue, that alpha comes out
premultiplied, that a clip actually stops paint, that a gradient runs the way round it was asked for.

## What is not bound yet

Path objects (`plutovg_path_t`) and the traversal callbacks, texture paints, the font-face cache and
its system-font loading, JPEG output, and image loading from files. The canvas API covers all of it in
the shapes a program usually wants; these are the parts that need more of the library's own object
model exposed, and none of them was needed to make the package useful.

## Licence

Three of them, which is a first for this org and is why `LICENSE` is longer than usual.

The sysl binding is **ISC**. PlutoVG itself is **MIT**. Its rasterizer and stroker are derived from
FreeType and carry the **FreeType Licence** (`FTL.TXT`), whose one live obligation is to credit
FreeType in the documentation of a product that uses it — this paragraph is that credit. The three
vendored `stb` headers are public domain.
