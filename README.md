# gl

OpenGL 3.3 core bindings for [Milo](https://github.com/milo-language/milo), plus a safe
layer over them. No dependencies beyond the standard library.

```bash
milo add github.com/milo-language/gl            # latest release
milo add github.com/milo-language/gl@v0.1.1     # or pin a specific tag
```

```milo
from "gl" import { Gpu, Shader, Mesh }

var sh = Shader.compile(VERT, FRAG)!
var quad = Mesh.fullscreenQuad()
Gpu.clear(0.0, 0.0, 0.0, 1.0, true)
sh.bind()
sh.uniformF("time", t)
quad.draw()
```

`gl` owns the shader, buffer, texture and framebuffer lifecycles and keeps the raw
pointers out of your program. Drop to `gl/raw` for an entry point it does not wrap.

## Handles are move-tracked

Every handle here is `@noCopy`, and `free` consumes it. Using one after it is freed, or
freeing it twice, is a **compile error** rather than a driver-level mystery:

```milo
let t = Texture2D.rgba8(w, h, pixels, false)
t.free()
t.bind(0)   // error: use of moved variable 't'
```

These types are integers, so without `@noCopy` the all-fields-Copy rule would make them
Copy and `free` would consume nothing. They deliberately have no `Drop`: deleting a GL
object needs the context that made it to still be current, so a destructor firing during
teardown is undefined behaviour rather than a leak. Forgetting `free` leaks; the two
errors worth catching are caught.

Uploads are bounds-checked too — the driver reads `w * h` elements off a pointer with no
idea how long your `Vec` is, so a short one would be a heap over-read from a call with no
`unsafe` at the call site.

## Textures for 3D, not just for full-frame passes

The constructors all start clamped, unfiltered and mip-less, which is right for a texture
that is a picture of the whole frame. A texture laid over 3D geometry wants the other
three:

```milo
let ground = Texture2D.srgb8(w, h, bytes, true)
ground.setWrap(Wrap.Repeat)      // UV is world position over a period, not 0..1
ground.generateMipmaps()         // after the upload — it derives the chain from level 0
ground.setAnisotropy(16.0)       // no-op without GL_EXT_texture_filter_anisotropic
```

`srgb8` takes three sRGB bytes per pixel — what a PNG decodes to — and the sampler
decodes to linear **in hardware, before filtering**. Doing it afterwards in the shader is
both slower and wrong: a bilinear tap averages four sRGB bytes, and the average of two
sRGB values is not the sRGB of their linear average, so edges come out too dark. A
lookup table per fetch has the same flaw and costs a dependent read.

Mipmaps are not optional for anything tiled across a 3D surface. Without them a distant
pattern samples one texel out of the dozen the pixel covers, and which one changes as the
camera moves — the ground crawls and glitters. Anisotropy then fixes what mips alone get
wrong at a grazing angle, where trilinear picks one level from the pixel's widest axis and
blurs the direction that was not compressed.

## darwin and linux only

`milo.json` declares `"targets": ["darwin", "linux"]`, and the compiler enforces it —
building for Windows names the package rather than failing on a missing symbol.
`opengl32.dll` exports GL 1.1 only, so every 3.3 entry point here would be undefined.

3.3 core is the floor on purpose: it is the highest version macOS ships, and old enough
that every Mesa and every driver of the last decade has it.

## A context is your job

Every call needs a current GL context, and creating one belongs to the window system, not
here — the library deliberately depends on nothing but the standard library. With SDL2
that is `SDL_GL_SetAttribute` + `SDL_WINDOW_OPENGL` + `SDL_GL_CreateContext`, which the
[`sdl`](https://github.com/milo-language/milo-sdl) package's `sdl/gl` module provides;
`examples/` and `tests/` use it, and they carry that dependency in their own manifests so
the published package does not. Calling into GL with no context bound is undefined
behaviour, not an error return.

## Verified bindings

Every declaration carries `@cSig`, so the signature is checked against the real GL header
at build time on any machine that has one — including each pointer parameter's pointee
width, which is what an out-param's contract actually is. A machine with neither
`OpenGL/gl3.h` nor `GL/glcorearb.h` gets a named warning, not a silent pass.

## License

MIT
