# Codebase notes

A map of tev's internals as they exist in this fork, written for whoever picks up the next feature. Line numbers refer to the tree at the merge commit `809b02c` plus our changes (2026-10-27); they drift with every upstream merge, so confirm with `rg` before editing.

Roughly 37k lines of C++ across `src/` and `include/`, plus ~32 vendored submodules.

## Build and layout

```
src/            core: viewer, canvas, image, colour, IPC
src/imageio/    one file per format, plus the loader registry
include/tev/    headers, mirroring src/
dependencies/   submodules, pinned by the superproject
resources/      icons, screenshot
```

Build and install:

```bash
cmake . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j
cmake --install build --prefix ~/.local/opt/lumina-inspector
ln -sf ~/.local/opt/lumina-inspector/bin/lumina-inspector ~/.local/bin/lumina-inspector
```

The binary is ~44 MB, unstripped. A clean build takes 10–20 minutes on 16 threads; an incremental rebuild of one file is seconds.

Submodule pointers are part of the commit. After merging upstream, always run `git submodule update --init --recursive`, or the build silently uses the wrong dependency revisions.

## Startup path

`src/main.cpp` parses arguments, constructs `ImageViewer` (a `nanogui::Screen`), and starts the IPC server. Images listed on the command line are handed to `ImagesLoader`, which loads them on the thread pool and posts results to the UI thread.

Single-instance behaviour is important and surprising: if another instance already listens on the configured host, the new process forwards its arguments over IPC and exits with status 0. This is why the fork uses port `14159` — otherwise launching it while stock tev is open silently hands the images to tev.

## Image loading

`ImageLoader::getLoaders()` in `src/imageio/ImageLoader.cpp` returns a static vector of loaders, tried in order until one accepts the stream. Adding a format is: write a class, add it to that list. There is no dynamic registration.

`ImageLoader::load` is a coroutine returning `Task<vector<ImageData>>`, because decoding runs on the worker pool. `ImageData` carries channels, layer names, the `toRec709` matrix, HDR metadata, `nativeMetadata`, and `renderingIntent`.

### OpenEXR specifics

`src/imageio/ExrImageLoader.cpp` reads the file through a memory-mapped `Imf::IStream`. Relevant behaviour:

- `chromaticities` is read when present and stored in `nativeMetadata.chroma`.
- `adoptedNeutral` and `whiteLuminance` are honoured.
- `renderingIntent` is forced to `AbsoluteColorimetric` for OpenEXR, on the argument that scene-linear data should not be white-point adapted.
- `colorInteropID` is **not** read. This is feature #1 in the wishlist.

## Colour pipeline

This is the part to understand before touching anything colour-related.

```
file pixels (native space)
   │
   ├─ loader sets ImageData::toRec709 (a 3×3 matrix) from chromaticities/CICP/ICC
   │
   ├─ Image constructor applies it on the CPU, once, at load
   │     Image.cpp: "tev assumes an image to be internally represented in sRGB/Rec709 space"
   │
   └─ every consumer sees Rec.709-linear values:
        ImageCanvas (display), CanvasStatistics (histogram), pixel probe, diff, save
```

Consequences:

- The shader never sees native AP0 values. Any display transform must either run on Rec.709-linear input, or the load-time conversion must be disabled.
- For linear EXR the conversion is a pure matrix, so it is invertible. A LUT bake can therefore prepend the inverse matrix and remain exact — this is the trick that lets feature #4 avoid touching the load path.
- The pixel probe and histogram apply a separate **inspection colour space** on top, which is configurable independently (see below). That is why probe values can be correct in AP0 even though the shader input is Rec.709.

## Display and shader

`src/UberShader.cpp` builds one large GLSL program (and a Metal variant for macOS) that does exposure/offset, tonemap, gamma, brightness limit, and highlight rolloff. `ETonemap` in `include/tev/Common.h` has only `None`/`Gamma`, `FalseColor`, and `PositiveNegative`.

There is no RRT, no ODT, no gamut compression. A single fixed hook before or after exposure is the natural place to sample a 3D LUT; that is the entire shader-side cost of feature #4.

Framebuffer initialisation is logged at startup, for example `primaries=bt2020 transfer=gamma22 range=wide_gamut_sdr`. If that line changes, colour behaviour changed.

## UI and input

`src/ImageViewer.cpp` is ~3k lines and builds the UI by hand with nanogui. Relevant points:

- `keyCallback` maps keys directly to actions in a long `if` chain (around line 1216). `Up`/`Down`/`W`/`S`/`PageUp`/`PageDown` change image; `Left`/`Right` change channel group. There is no keymap.
- The **Colors** popup (around line 377) contains the inspection colour space: primaries (with a Custom entry and eight editable chromaticity boxes), transfer, white-point adaptation, premultiplied alpha. AP0 readout is already possible here without any code change.
- `updateTitle` builds the window caption including pixel coordinates and values.
- `HelpWindow` renders the About panel and shortcut list.

nanogui is a fork (`dependencies/nanogui`) and has a `Graph` widget for simple line plots. tev's `MultiGraph` is a histogram adapted from it and is the closest existing thing to a plot panel.

## IPC

`include/tev/Ipc.h` and `src/Ipc.cpp`. Framing:

```
[uint32_t length][uint8_t type][payload]
```

Little-endian. Type IDs are append-only; new behaviour gets a new ID rather than a changed payload (`UpdateImageV2`, `V3`, `OpenImageV2` are the precedent).

| ID | Operation |
|---|---|
| 0 | OpenImage |
| 1 | ReloadImage |
| 2 | CloseImage |
| 3 | UpdateImage |
| 4 | CreateImage |
| 5 | UpdateImageV2 |
| 6 | UpdateImageV3 |
| 7 | OpenImageV2 |
| 8 | VectorGraphics |

`UpdateImage` takes raw float channels with explicit offsets and strides, so a script can push arbitrary computed images. There is no query or command channel yet; that is feature #6.

Default host is `127.0.0.1:14159` in this fork (`src/Ipc.cpp`, around line 397).

## Vector graphics

`include/tev/VectorGraphics.h` defines `VgCommand`, a serialisable subset of NanoVG: paths (`MoveTo`, `LineTo`, `QuadTo`, `BezierTo`, `Arc`, `Circle`, `Ellipse`, `Rect`, `RoundedRect`), state (`Save`/`Restore`), paint (`FillColor`, `StrokeColor`, `StrokeWidth`), and text (`Text`, `FontFace`, `FontSize`, `TextAlign`).

Flow: IPC packet → `ImageViewer::updateImageVectorGraphics` → `Image::updateVectorGraphics(append, commands)` → stored on the image → `ImageCanvas::draw` replays them.

Coordinates are **image pixels**, mapped through `displayWindowToNanogui`, so overlays zoom and pan with the image. They are drawn for the current image and for the reference. This is a complete overlay system that is barely used upstream, and it is how feature #10 works without any C++ change.

## Statistics

`ImageCanvas::canvasStatistics()` returns a cached `CanvasStatistics`, keyed by inspection parameters, image, channel group, reference, metric, and crop. Computation runs on the worker pool and is invalidated when an image changes. The histogram is drawn by `MultiGraph`.

Per-image min/max/mean/clip percentages are not exposed; feature #8 builds on this machinery.

## Settings

There are none. No config directory is created, and no state survives a restart. Feature #3 is greenfield.

## Measured performance

On a 16-thread desktop with an RTX 3080, Wayland:

| Case | Result |
|---|---|
| 6 × 3040×2014 EXR | 0.13 s total |
| 1 × 3040×2014 EXR | 9 ms |
| 122 images, 2.6 GB on disk | 3.3 s load, ~7.9 GB RSS |

Load time is dominated by decode and conversion, and everything stays resident. There is no eviction.

## Gotchas

- **Port collision.** Stock tev and this fork must not share an IPC port; they now differ (`14158` vs `14159`). If a launch "does nothing", check whether another instance owns the port.
- **`-r` recurses into everything**, including JSON and Markdown, and logs an error per file. Prefer an explicit file list until feature #5.
- **RAM.** The resident set is roughly three times the on-disk size for compressed EXR. A large folder will exhaust memory rather than slow down.
- **GCC 16 warnings** from DirectXTex are expected and harmless.
- **`--version` reports `2.14.0dev`** because the tree is ahead of the last tag. Not a bug.
- **Submodules** must be updated after every upstream merge.
- **Do not reformat upstream files.** A whitespace diff will conflict with every future merge.

## Where to add things

| Goal | Start here |
|---|---|
| New image format | `src/imageio/`, then the list in `src/imageio/ImageLoader.cpp` |
| New CLI flag | `src/main.cpp` (`ArgumentParser` block) |
| New key binding | `src/ImageViewer.cpp`, `keyCallback` |
| New display effect | `src/UberShader.cpp` plus a new file for the state |
| New IPC operation | `include/tev/Ipc.h`, `src/Ipc.cpp`, dispatch in `src/main.cpp` |
| New statistic | `src/ImageCanvas.cpp`, `CanvasStatistics` |
| New panel | `src/ImageViewer.cpp`, or a new widget file |
