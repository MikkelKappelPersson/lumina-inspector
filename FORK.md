# lumina-inspector

A fork of [tev — the EDR viewer](https://github.com/Tom94/tev), developed as a scientific and technical image analysis tool for HDR and floating-point data.

## Base

- Upstream: <https://github.com/Tom94/tev>
- Forked at tag `v2.14.0`: `18dabdd78172fea9aba8740ee33e90af1de9f1e4` (2026-08-22)
- Upstream master merged: `809b02c` (2026-10-03)

`upstream` points at `Tom94/tev`. The local `master` branch mirrors upstream; `main` is this fork.

## Upstream merges

| Date | Upstream commit | Note |
|---|---|---|
| 2026-10-27 | `809b02c` | 21 commits past `v2.14.0`; adds KTX/KTX2 support, KTX channel handling, CI fixes |

## Branch model

Soft fork. `main` carries our changes on top of the pinned release. Upstream remains the source of truth for everything we have not touched, so bug fixes and new formats stay cheap to pick up.

To pull upstream changes:

```bash
git fetch upstream
git merge upstream/master
```

Merge once the change set is large; rebase only while the delta is still a handful of commits.

## Rules for changes

1. New features go in new files. Touch upstream files only at registration seams.
2. Never reformat or rename upstream code. Format only what is added.
3. One concern per commit, with the reason in the message.
4. Keep `main` buildable at every commit.

These rules exist so that merging upstream stays an afternoon, not a week.

## Building

Requires CMake, Ninja, a C++23 compiler, `nasm`, and the X11/Wayland development packages.

```bash
cmake . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j
cmake --install build --prefix ~/.local/opt/lumina-inspector
ln -sf ~/.local/opt/lumina-inspector/bin/lumina-inspector ~/.local/bin/lumina-inspector
```

The binary is named `lumina-inspector`, so the stock AppImage stays available as `tev`. The fork listens on IPC port `14159` rather than tev's `14158`, so the two never capture each other's launches.

## Documentation

- [`docs/features.md`](docs/features.md) — feature wishlist and handoff notes, with effort and verification steps.
- [`docs/codebase-notes.md`](docs/codebase-notes.md) — where things live in tev, the extension seams, and the traps.

## Licence

GPL-3.0, inherited from tev. See `LICENSE.txt` for the full text and `NOTICE` for the attribution summary. This repository is the complete corresponding source for any binary built from it.

## Planned changes

- [ ] Read EXR `colorInteropID` so ACES / AP0 files display with correct primaries
- [ ] Remappable keys; `Left` / `Right` flip images
- [ ] Persist settings between sessions (inspection colour space, exposure, gamma)
- [ ] Display transform via a 3D LUT baked from an OCIO config
- [ ] Folder grouping and natural sort
- [ ] Live plot overlays over IPC, driven by external scripts

Everything not on this list is intended to live outside the binary, driven over the IPC socket, so the fork stays small.
