# gl

OpenGL 3.3 core bindings for [Milo](https://github.com/milo-language/milo), plus a safe
layer over them. No dependencies beyond the standard library.

```bash
milo add github.com/milo-language/gl            # latest release
milo add github.com/milo-language/gl@v0.1.0     # or pin a specific tag
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

## darwin and linux only

`milo.json` declares `"targets": ["darwin", "linux"]`, and the compiler enforces it —
building for Windows names the package rather than failing on a missing symbol.
`opengl32.dll` exports GL 1.1 only, so every 3.3 entry point here would be undefined.

3.3 core is the floor on purpose: it is the highest version macOS ships, and old enough
that every Mesa and every driver of the last decade has it.

## A context is your job

Every call needs a current GL context, and creating one belongs to the window system, not
here — this package deliberately does not depend on SDL, GLFW, or any windowing library.
With SDL2 that is `SDL_GL_SetAttribute` + `SDL_WINDOW_OPENGL` + `SDL_GL_CreateContext`.
Calling into GL with no context bound is undefined behaviour, not an error return.

## Verified bindings

Every declaration carries `@cSig`, so the signature is checked against the real GL header
at build time on any machine that has one — including each pointer parameter's pointee
width, which is what an out-param's contract actually is. A machine with neither
`OpenGL/gl3.h` nor `GL/glcorearb.h` gets a named warning, not a silent pass.

## License

MIT
