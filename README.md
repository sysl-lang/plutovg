# plutovg

2D vector graphics for sysl, bound to [PlutoVG](https://github.com/sammycage/plutovg) — paths,
filling, stroking, dashing, gradients, clipping, transforms and text, rasterized into memory.

```
dependencies {
  plutovg { git = "github.com/sysl-lang/plutovg", version = "0.2.1" }
}
```

```sysl
import sh.sysl.plutovg

val s = surface(200, 200).unwrap()
val cv = canvas(s).unwrap()

cv.rgb(0.1, 0.2, 0.9)
cv.circle(100.0, 100.0, 60.0)
cv.fill()

s.write_png("circle.png")
```

Nothing is closed and nothing is linked: the C is carried here, so `sysl build .` needs no flags at
all.

## It draws into memory and nothing else

What happens to those pixels afterwards — a PNG, a window, a 320×480 SPI panel — is somebody else's
problem. That is what makes this package independent of any display and of any microcontroller, and
testable on a desktop with no hardware at all.

A surface holds **premultiplied ARGB32**, which on a little-endian machine is the bytes B, G, R, A.
`data()` hands the whole buffer back.

## Why PlutoVG

The header was read rather than the README trusted, and the numbers are unusually good for a binding:

- **Every parameter is a `float`, an `int`, a pointer or a `const T*`.** **So this package has no
  `shim.c` at all** — the entire library is reachable by declaring it.
- **`float` throughout, not `double`** — which matters on a Cortex-M33, whose FPU is single-precision
  and which would otherwise emulate every coordinate in software.
- **`plutovg_surface_create_for_data`** — the caller owns the pixels. The house pattern, available
  straight off.
- **C99, no external dependencies**, 11 `.c` and 10 `.h` files, all vendored here.
- It carries its own TrueType rasterizer, so **text needs no second binding**.

## Nothing is closed by hand

`Surface`, `Canvas` and `Font` each hold a C allocation, each is reached through `&T`, and each has an
`impl Drop` that releases it when the last reference goes. **There is no `close` in this API.**
PlutoVG's objects are reference counted and so is `&T`, so the two agree — and they agree about more
than your own variables:

```sysl
val cv = canvas(surface(200, 200).unwrap()).unwrap()
```

`plutovg_canvas_create` takes a count on the surface it draws into, so the canvas holds it alive even
though nothing here named it. The previous shape of this package asked you to call `close` in the
right order and warned about it in a comment.

Storage lent by `for_data` is yours, and releasing the surface does not free it.

## A constructor answers an Option

`surface`, `for_data`, `canvas`, `font` and `font_from_data` all allocate, and each answers `None`
when it cannot. That is not a formality on the machine this package exists for — a 600 KiB surface on
a microcontroller is exactly the request that fails — and `font` answers `None` for the ordinary case
of a file that is not there. So the failure is in the type rather than in an `ok()` you can forget to
ask.

## Every enumeration is an enum

`operator` takes an `Operator`, `line_cap` takes a `LineCap`, `fill_rule` takes a `FillRule`. There is
no way to pass a spread method where a line join belongs, or a bare `3`:

```sysl
cv.line_cap(LineCap.Round)
cv.fill_rule(FillRule.EvenOdd)

print(Operator.SrcOver)     -- "src-over", not "3"
```

**None of them carries an `Other` arm**, unlike the enums in `sh.sysl.cairo`, and the difference is
PlutoVG's rather than a matter of taste: it never *answers* one of these. Every one is written to the
canvas and none is read back, so there is no unknown value to keep — which leaves a plain enum with
`Eq` for free and no way to spell a meaningless one.

**No number is written anywhere in this package.** Each is what the C compiler computes for PlutoVG's
own name, out of the header vendored beside it:

```sysl
c const
    OPERATOR_SRC_OVER: int = "PLUTOVG_OPERATOR_SRC_OVER"
    LINE_CAP_ROUND:    int = "PLUTOVG_LINE_CAP_ROUND"
```

That is `15 §7`'s answer to a transcribed constant being "correct on one machine" with "nothing
checking it" — and it costs nothing here, because the header is carried in the package. It found
something immediately: the hand-typed list had **eight** compositing operators and PlutoVG has
**twelve**. `DstOut`, `SrcAtop`, `DstAtop` and `Xor` were not a decision anybody made.

## Everything that is C lives in one place

`sh/sysl/plutovg/c/` — the eleven `.c` files, their headers, and one file of declarations over them,
which nothing outside the package is meant to call. The module above it turns those into sysl and is
what an application imports.

The name is the point. A call site reads `c.canvas_destroy(h)`, so crossing into C is visible without
a comment:

```sysl
struct Surface
    handle: *c.Surface

    width(&self) -> int = c.surface_get_width(self.handle)
```

`c.Surface`, `c.Canvas` and `c.FontFace` are `opaque struct`s rather than `*u8`, so a canvas cannot be
handed to a function that wants a surface. That costs nothing at run time.

## Rendering for a small machine

`for_data` borrows storage the caller already owns rather than allocating it:

```sysl
var band: [80 * 1024]u8
val s = for_data(band[..], 320, 64, 320 * 4).unwrap()
```

A full 320×480 surface at 32bpp is **600 KiB**, more than an RP2350 has. A 320×64 band is 80 KiB,
which fits — so a program renders a band, pushes it, and reuses the buffer.

`sh.sysl.st7796` is the other half of that arrangement: it takes an ARGB32 band and converts to RGB565
as it pushes. **The conversion lives there rather than here**, so this stays a faithful binding, and
that is the only place the two packages touch.

## Capabilities

`heap` and `posix`, and both are narrower than they look.

**`heap`** is honest and not yet avoidable: PlutoVG mallocs its canvases, path storage and glyph
caches, and unlike QOI it has no `PLUTOVG_MALLOC` hook to redirect. `for_data` keeps the *pixels* out
of the heap, which is the large allocation and the reason band rendering is practical, but the canvas
itself still allocates.

**`posix` is `write_png` and `font`, and nothing else.** The whole drawing surface touches no file
system. A program that renders into its own buffer, loads a font from bytes linked into the binary
with `font_from_data`, and never writes a PNG uses no operating system at all — the capability is
declared because those two functions are part of the module, and a capability is a property of the
module rather than of the call.

## Tests

```
sysl test .
```

Thirty-eight, and **they are pixel tests**. A binding can be wrong in a way that compiles, links and
returns without complaint — a parameter in the wrong order, a float read as a double, an enum off by
one — and the only thing that catches it is looking at what was drawn. So most of these rasterize
something small and read the bytes back: that red is red and not blue, that alpha comes out
premultiplied, that a clip actually stops paint, that a gradient runs the way round it was asked for.

Two of them exist to check that an enum *reaches* the library rather than merely type-checking: the
fill rule decides whether an overlap is a hole, and a round cap paints past the end of a line where a
butt cap does not. A `code()` returning the wrong number paints the wrong picture there and nowhere
else. A third asks PlutoVG for its own reference count and checks that making a canvas raises the
surface's by one — the claim above, measured rather than asserted, and as a delta because the absolute
number is the library's business.

## Images

```
val logo = load_image("logo.png").unwrap()      // PNG, JPEG, BMP, PSD, TGA, GIF, HDR, PIC, PNM
val icon = image_from_data(bytes).unwrap()      // the same, from memory

cv.save()
cv.translate(f32(x), f32(y))
cv.scale(f32(w) / f32(logo.width()), f32(h) / f32(logo.height()))
cv.texture(logo)
cv.rect(0.0, 0.0, f32(logo.width()), f32(logo.height()))
cv.fill()
cv.restore()
```

**The decoder is stb_image, vendored beside the library**, so an image costs nothing linked that the
package did not already carry — which is why loading belongs here rather than in a caller.

**A texture is a paint source and not a drawing operation**, exactly like a colour or a gradient:
nothing appears until something is filled with it, and *the shape that is filled is the shape the
image is cut to*. That is what makes a round-cornered image a `round_rect` and not a feature.

**It takes no matrix, deliberately.** The surface is laid down in user space, one pixel to one unit,
so where it lands and how big it is are the canvas's own `translate` and `scale` — two ways of saying
that would disagree the first time somebody used both. `TextureType.Tiled` is the other half:
a plain texture is transparent past its own edge, a tiled one starts again.

## What is not bound yet

Path objects (`plutovg_path_t`) and the traversal callbacks, the paint object (`plutovg_paint_t`) and
the explicit texture matrix, the font-face cache and its system-font loading, JPEG output, and
base64 image data. The canvas API covers all of it in the shapes a program usually wants; these are
the parts that need more of the library's own object model exposed, and none of them was needed to
make the package useful.

**Loading a real typeface has no test**, only the two refusals. It would need a font file, and a
megabyte of somebody else's copyright to assert one number about is not a trade worth making — so
`font` and `font_from_data` are exercised on the failing path and read on the succeeding one.

## Licence

Three of them, which is a first for this org and is why `LICENSE` is longer than usual.

The sysl binding is **ISC**. PlutoVG itself is **MIT**. Its rasterizer and stroker are derived from
FreeType and carry the **FreeType Licence** (`FTL.TXT`), whose one live obligation is to credit
FreeType in the documentation of a product that uses it — this paragraph is that credit. The three
vendored `stb` headers are public domain.
