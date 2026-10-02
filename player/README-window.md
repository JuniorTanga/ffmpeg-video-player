# Embedding the player in an existing SDL2 window

`player-sdl-window.c` is a fork of [`player-sdl.c`](player-sdl.c) adapted so the
player renders into an SDL2 window that the host application already owns,
instead of creating and destroying its own.

The entry point changes from `main()` to:

```c
int ffplay(int argc, char* argv[], SDL_Window* _window);
```

Declare it in `ffplay-window.h`. Typical use from an SDL2 application:

```c
SDL_Window*   window   = SDL_CreateWindow("My app", ...);
SDL_Renderer* renderer = SDL_CreateRenderer(window, -1, ...);

char* argv[] = { "ffplay", "-x", "800", "-y", "400", "-autoexit", "intro.mp4" };
ffplay(7, argv, window);
```

The call blocks until playback finishes and then returns, leaving the caller's
window and renderer intact.

## What changed relative to `player-sdl.c`

| Change | Reason |
| --- | --- |
| `int main(int argc, char* argv[])` → `int ffplay(int argc, char* argv[], SDL_Window* _window)` | turns the executable entry point into a library call |
| `window = _window` instead of `SDL_CreateWindow(...)` | reuse the host's window |
| `renderer = SDL_GetRenderer(window)` instead of `SDL_CreateRenderer(...)` | reuse the host's renderer |
| `SDL_Init(sdl_flags)` → `SDL_InitSubSystem(sdl_flags)` | the host already called `SDL_Init`, re-initialising would reset its state |
| `SDL_Quit()` and `exit(0)` disabled; `SDL_DestroyWindow`/`SDL_DestroyRenderer` commented out | the host owns the window, renderer and SDL lifetime |
| new `static int exit_` flag, `for (;;)` loop changed to `while (!exit_)` | return control to the caller at end of stream instead of killing the process |
| `SDL_SetWindowTitle`, `SDL_SetWindowSize`, `SDL_SetWindowPosition` commented out, `SDL_GetWindowSize` used instead | the player must not resize or retitle a window it does not own |
| `av_codec_next()` → `av_codec_iterate()` | `av_codec_next` is deprecated and removed in newer FFmpeg |
| `fopen()` → `fopen_s()` | MSVC compatibility |
| `_CRT_SECURE_NO_WARNINGS` defined | MSVC compatibility |

All of this is GPL-3.0, inherited from the upstream project. Keep the same
license and attribution when redistributing.

## Building

Added to `player/CMakeLists.txt` as the `player-sdl-window` target:

```bash
cmake -S player -B build -DFFMPEG_ROOT=/usr/local -DSDL2_ROOT=/usr
cmake --build build
```

`FFMPEG_ROOT` and `SDL2_ROOT` point at development installations that contain
the headers and import libraries. At runtime the shared libraries
(`avformat`, `avcodec`, `avdevice`, `avfilter`, `avutil`, `swscale`,
`swresample`, `postproc` and `SDL2`) must be reachable, either through the
system loader path or copied next to the executable.

FFmpeg 4.3 or newer is required for `av_codec_iterate`. The original
`player-sdl.c` still builds against older FFmpeg because it does not carry that
change.