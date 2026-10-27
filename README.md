# Lumina Inspector

A scientific and technical image analysis tool for high dynamic range and floating-point image data.

Lumina Inspector is a fork of [tev — The EDR Viewer](https://github.com/Tom94/tev). It keeps tev's speed, format coverage, and colour accuracy, and adds what quantitative work needs: source colour-space awareness, colour-managed display transforms, persistent settings, and a scriptable overlay channel for measurements and plots.

> **Status:** early fork, tracking tev. Everything tev does, it still does. Additions are listed under [What is different](#what-is-different-from-tev).

![Screenshot](resources/screenshot.png)
_The interface inherited from tev: a false-colour comparison of two multi-layer OpenEXR images. Image courtesy of [openexr-images](https://github.com/openexr/openexr-images)._

## Why this fork exists

Scientific images are data, not pictures. The questions you ask of them are quantitative — what is this pixel's value, which colour space is this in, how far is this render from the reference, where does it clip — and a viewer built for photography answers only some of them.

Lumina Inspector is aimed at that:

- **Values, unclamped.** Float data is read and reported as stored, including values far outside `[0,1]` and negative values from out-of-gamut conversions.
- **Colour as metadata, not assumption.** Source primaries, transfer function, and white point are treated as properties of the data, and the display transform is a separate, explicit step.
- **Measurements over pictures.** Histograms, error metrics against a reference, and script-driven overlays for regions, annotations, and plots.
- **Reproducibility.** Settings and transforms are files, not hidden state, so a figure can be regenerated from a recorded configuration.

It is not a compositor, a video player, or an asset manager, and it does not try to be.

## What is different from tev

Current:

- Installs as `lumina-inspector`, so the stock `tev` can stay installed alongside it.
- IPC listens on port `14159` instead of `14158`, so the two never capture each other's launches.
- Own window title and About panel.

Planned:

- [ ] Read EXR `colorInteropID` so ACES / AP0 files display with correct primaries without manual tagging
- [ ] Colour-managed display transform via a 3D LUT baked from an OCIO config
- [ ] Remappable keys; `Left` / `Right` flip images
- [ ] Persistent settings between sessions (inspection colour space, exposure, gamma)
- [ ] Folder grouping and natural sort
- [ ] Measurement and plot overlays driven over IPC

## Installation

Linux, from source. This is the only supported path today.

```bash
git clone --recursive https://github.com/MikkelKappelPersson/lumina-inspector
cd lumina-inspector
cmake . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j
cmake --install build --prefix ~/.local/opt/lumina-inspector
ln -sf ~/.local/opt/lumina-inspector/bin/lumina-inspector ~/.local/bin/lumina-inspector
```

Requirements: CMake, a C++23 compiler, NASM, and the X11 / Wayland / OpenGL / D-Bus development packages. On Arch Linux:

```bash
sudo pacman -S cmake ninja nasm wayland-protocols libxkbcommon dbus libffi glu libglvnd \
    libx11 libxinerama libxcursor libxrandr libxi
```

On Debian/Ubuntu the upstream list applies:

```bash
sudo apt-get install cmake libdbus-1-dev libffi-dev libglu1-mesa-dev libwayland-dev \
    libxkbcommon-dev nasm wayland-protocols xorg-dev
```

## Usage

### Graphical interface

Open images with the file dialog or by dragging them in. They can be reloaded, closed, or filtered at any time, so there is no harm in opening more than you need.

Left-click selects an image; right-click selects a reference to compare against. `Up` / `Down` move the selection. Press `h` for the full shortcut list, or hover any control for a tooltip.

### Command line

```bash
lumina-inspector foo.exr bar.exr
```

Layers and channels can be selected per image:

```bash
lumina-inspector :depth foo.exr :r,g,b foo.exr bar.exr
```

For all options:

```bash
lumina-inspector -h
```

### Over the network

The viewer can be driven over a simple TCP protocol. The `--host` argument sets the address to listen on; the default is `127.0.0.1:14159`.

| Operation | Function |
| :--- | :--- |
| `OpenImage` | Opens an image from a path on the machine the viewer runs on |
| `CreateImage` | Creates a blank image with a given name, size, and channels |
| `UpdateImage` | Updates pixels in a rectangular region |
| `CloseImage` | Closes a named image |
| `ReloadImage` | Reloads an image from disk |
| `VectorGraphics` | Draws vector graphics over a named image |

Each packet is:

```
[uint32_t total_length_in_bytes][byte operation_type][byte[] operation_specific_payload]
```

with little-endian integers. `src/Ipc.cpp` contains the `IpcPacket::set*` helpers that show exactly how each packet is assembled, with no external dependencies.

Existing tev clients work once pointed at the right port:

- [Python](https://pypi.org/project/tevclient/): `pip install tevclient`
- [C++](https://github.com/westlicht/tevclient)
- [Rust](https://crates.io/crates/tev_client)

## File formats

- **EXR** (via [OpenEXR](https://github.com/AcademySoftwareFoundation/openexr))
- **JPEG XL** (including gain maps; via [libjxl](https://github.com/libjxl/libjxl))
- **JPEG XR** (via [jxrlib](https://github.com/Tom94/jxrlib))
- **JPEG** (including gain maps; via [libjpeg-turbo](https://github.com/libjpeg-turbo/libjpeg-turbo))
- **JPEG 2000** (via [openjpeg](https://github.com/uclouvain/openjpeg))
- **PNG**, APNG (via [libpng](https://github.com/pnggroup/libpng))
- **PFM**, PAM, PNM, PBM, PGM, PPM (see [NetPBM](https://netpbm.sourceforge.net/) and [PFM](https://www.pauldebevec.com/Research/HDR/PFM/))
- **QOI** (via [qoi](https://github.com/phoboslab/qoi))
- **DDS** (via [DirectXTex](https://github.com/microsoft/DirectXTex) + [DirectXMath](https://github.com/microsoft/DirectXMath) + [DirectX-Headers](https://github.com/microsoft/DirectX-Headers))
- **KTX2**, **KTX** (via [KTX-Software](https://github.com/KhronosGroup/KTX-Software) + [bcdec](https://github.com/iOrange/bcdec) + [etcdec](https://github.com/iOrange/etcdec))
- **WEBP** (via [libwebp](https://chromium.googlesource.com/webm/libwebp))
- **TIFF**, **DNG** (via [libtiff](https://gitlab.com/libtiff/libtiff))
- **AVIF** (including gain maps; via [dav1d](https://github.com/videolan/dav1d) + [libheif](https://github.com/strukturag/libheif))
- **HEIC** (including gain maps; via [libde265](https://github.com/strukturag/libde265) + [libheif](https://github.com/strukturag/libheif); disabled by default, enable with the `TEV_SUPPORT_HEIC` CMake option and check patent law in your jurisdiction)
- **BMP**, ICO, CUR (including colour profiles)
- **HDR**, GIF, PIC, PSD, TGA (via [stb_image](https://github.com/wjakob/nanovg/blob/master/src/stb_image.h))
- **RAW** and camera-specific RAW formats (via [LibRaw](https://github.com/LibRaw/LibRaw))
- **DICOM** (via [GDCM](https://github.com/malaterre/GDCM))
- **FITS** (via [cfitsio](https://github.com/HEASARC/cfitsio.git))

## Attribution and licence

Lumina Inspector is an independent fork of **tev — The EDR Viewer**.

- Upstream: <https://github.com/Tom94/tev>
- Upstream copyright: Copyright (C) 2017–2026 Thomas Müller and contributors
- Upstream licence: GNU General Public License v3.0
- Fork modifications: Copyright (C) 2026 Mikkel Kappel Persson

All upstream code remains under the GPL-3.0. Modifications in this repository are released under the same licence. The full licence text is in [`LICENSE.txt`](LICENSE.txt); the attribution summary is in [`NOTICE`](NOTICE).

This project is not affiliated with, or endorsed by, the tev project. If you want the original, use [tev](https://github.com/Tom94/tev).

Older versions of tev (v1.30 and earlier) were released under the BSD 3-Clause licence. This fork is based on GPL-3 code and does not include that older history's licensing.
