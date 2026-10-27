# Feature wishlist

A handoff document for continuing work on Lumina Inspector. Each entry states what the feature is, why it matters, how to approach it, which files are involved, how to verify it, and how much it costs.

Status values: `idea`, `specified`, `in progress`, `done`, `deferred`, `rejected`.

Effort is calendar time for one person working part-time, not lines of code.

## Context

Lumina Inspector is a soft fork of [tev](https://github.com/Tom94/tev), pinned at `v2.14.0` and merged with upstream master through `809b02c` on 2026-10-27. The user-facing identity is the fork's; the C++ namespace, CMake project, and file headers remain upstream's so that merges stay cheap.

What the fork already changes:

- Binary and install name `lumina-inspector`, so stock `tev` can coexist.
- IPC on port `14159` instead of `14158`, so the two never attach to each other's instance.
- Window title, About panel, CLI header.
- A dedicated image browser panel on the right, holding the open-image list, find field, and playback controls.
- `README.md`, `NOTICE`, and this documentation.

Everything else is upstream tev. That is deliberate.

## Priorities at a glance

| # | Feature | Phase | Effort | Status |
|---|---|---|---|---|
| 1 | EXR `colorInteropID` support | 1 | half day | specified |
| 2 | Remappable keys, `Left`/`Right` flip | 1 | half day | specified |
| 3 | Settings persistence | 1 | 1–2 days | specified |
| 4 | Colour-managed display transform via 3D LUT | 1 | ~1 week | specified |
| 5 | Folder grouping and natural sort | 1 | 1–2 days | specified |
| 6 | IPC command and query channel | 2 | 3–4 days | idea |
| 7 | Live difference view | 2 | 2–3 days | idea |
| 8 | Per-image metrics table and CSV export | 2 | 3–4 days | idea |
| 9 | Batch export through the display transform | 2 | 1 day | idea |
| 10 | Measurement and plot overlays from scripts | 2 | client only | idea |
| 11 | Streaming / prefetch cache | 3 | 2–3 weeks | deferred |
| 12 | Ratings, tags, and filtering | 3 | ~1 week | deferred |
| 13 | Full OpenColorIO linkage | 4 | 2–4 weeks | deferred |

Phase 1 is the "daily friction" set. Phase 2 makes the tool analytical rather than merely visual. Phase 3 is about scale, and should not start until the corpus actually outgrows RAM. Phase 4 is a product decision, not a maintenance task.

---

## Phase 1 — correctness and ergonomics

### 1. EXR `colorInteropID` support

**Status:** specified · **Effort:** half day

**Why.** OpenEXR 3.x defines a string attribute `colorInteropID` that names the colour space of the pixel data. ACES-based pipelines write it; `lin_ap0_scene`, `lin_ap1_scene`, `lin_rec709_scene`, `lin_p3d65_scene`, `lin_rec2020_scene`, and `lin_adobergb_scene` are the six linear, scene-referred identifiers with defined chromaticities.

tev reads the older `chromaticities` attribute but not `colorInteropID`. Files carrying only the modern identifier therefore fall back to Rec.709 and display with the wrong primaries. This is the single most common reason ACES renders look wrong on load.

**Current behaviour.** `src/imageio/ExrImageLoader.cpp` checks `Imf::hasChromaticities(part.header())` and assigns `data.nativeMetadata.chroma` from it. If the attribute is absent, `nativeMetadata.chroma` stays at its default, which `Image` treats as Rec.709. The loader also forces `ERenderingIntent::AbsoluteColorimetric` for OpenEXR, deliberately: scene-linear data should not be white-point adapted. That is correct for `chromaticities`, but AP0's white point is D60, so an AP0 image with absolute intent renders slightly warm on a D65 display.

**Approach.**

1. In the EXR loader, when `chromaticities` is absent, look for a `StringAttribute` named `colorInteropID` and map the six known identifiers to `chroma_t` values. Add the AP0 and AP1 chromaticities to `src/Colors.cpp` next to the existing `rec709Chroma()` / `bt2020Chroma()` helpers.
2. For identifiers that denote a scene-referred space with a non-D65 white point, set `ERenderingIntent::RelativeColorimetric` so the white point is adapted to the display. This is a judgement call: it changes the "objective" behaviour the upstream comment argues for, so record the reasoning in the commit message.
3. Unknown identifiers fall through to the existing behaviour. Never guess.

**Files.** `src/imageio/ExrImageLoader.cpp`, `src/Colors.cpp`, `include/tev/Colors.h`.

**Verification.** Open an AP0 EXR with no `chromaticities` attribute and confirm the display matches the same file after tagging it with AP0 chromaticities. Hover a saturated pixel and confirm the inspection readout matches the file's stored values. Open an AP1 file and confirm it differs from the AP0 file in the expected direction. Open a Rec.709 EXR and confirm nothing changed.

**Risks.** Attribute spelling varies in the wild (`colorInteropID` is the OpenEXR name). Some writers also emit `chromaticities` alongside; when both exist, `chromaticities` wins, which is what upstream already does.

---

### 2. Remappable keys, `Left`/`Right` flip

**Status:** specified · **Effort:** half day for the flip, plus the config work in #3

**Why.** Image navigation is `Up`/`Down`, `W`/`S`, `PageUp`/`PageDown`. `Left`/`Right` changes channel groups. Every other image viewer uses left/right for navigation, and the keys are hardcoded in a long `if` chain, so there is no way to change them without editing C++.

**Current behaviour.** `ImageViewer::keyCallback` in `src/ImageViewer.cpp` (around lines 1216–1260 in the pinned tree) maps keys directly to actions. There is no keymap, no command registry, and no config file.

**Approach.** Two options, in increasing cost:

- **Minimal:** add `GLFW_KEY_LEFT` / `GLFW_KEY_RIGHT` as additional triggers for previous/next image, and move channel-group switching to `Ctrl+Left` / `Ctrl+Right`. Roughly ten lines. Solves the complaint, keeps the hardcoding.
- **Proper:** introduce a small command table (name → lambda) and a keymap loaded from the settings file. This is the same machinery that scripting and IPC command dispatch will want later, so it is worth doing once.

Do the minimal version first if the goal is to unblock daily use; schedule the proper version with #3.

**Files.** `src/ImageViewer.cpp`, `include/tev/ImageViewer.h`, plus a new keymap/config file.

**Verification.** Launch, press `Left`/`Right`, confirm image selection changes and channel groups are reachable another way. Confirm the shortcut overlay (`h`) reflects reality.

**Risks.** `Left`/`Right` also drive the reference image when a modifier is held; keep that behaviour reachable.

---

### 3. Settings persistence

**Status:** specified · **Effort:** 1–2 days

**Why.** Nothing is remembered between sessions. The inspection colour space, exposure, gamma, window geometry, and last-opened folder all reset on every launch. For a tool where a fixed inspection space is part of the method, that is a correctness problem, not a convenience one.

**Current behaviour.** There is no config directory at all. Verified after repeated launches: no `~/.config/tev`, no `~/.config/lumina-inspector`, no settings file anywhere.

**Approach.** Add a single JSON or TOML file under `$XDG_CONFIG_HOME/lumina-inspector/`. Load it at startup, save on exit and on change (debounced). Start with the small set that matters:

- inspection primaries, transfer, white-point adaptation, premultiplied alpha
- exposure, gamma, offset, tonemap mode
- window size and maximised state
- last directory

Keep the file schema versioned from the first commit; unknown keys are ignored, missing keys take defaults. Do not serialise the whole `ImageViewer`.

**Files.** New `src/Settings.cpp` + `include/tev/Settings.h`, with small call sites in `main.cpp` and `ImageViewer.cpp`.

**Verification.** Set AP0 inspection, exposure +2, quit, relaunch, confirm all three survive. Corrupt the file deliberately and confirm the app starts with defaults rather than failing.

**Risks.** Upstream has no settings layer, so this is the first real structural addition. Keep it isolated: a `Settings` object passed in, not globals.

---

### 4. Colour-managed display transform via 3D LUT

**Status:** specified · **Effort:** ~1 week

**Why.** tev's display path is primaries matrix → exposure → gamma, brightness limit, rolloff. There is no OpenColorIO and no filmic output transform. An ACES 2.0 output transform is not a matrix plus gamma, so what you see in the viewer does not match what a colour-managed application produces from the same file. For a tool that claims colour accuracy, that is the largest remaining gap.

**Approach.** Bake the transform to a 3D LUT and sample it in the shader. Do not link OpenColorIO yet.

1. Bake with `ociobakelut` from any OCIO config, for example an ACES 2.0 studio config:
   ```bash
   ociobakelut --iconfig studio-config.ocio \
     --inputspace "ACES2065-1" \
     --displayview "sRGB - Display" "ACES 2.0 - SDR 100 nits (Rec.709)" \
     --format spi3d --cubesize 65 ap0_to_srgb.spi3d
   ```
2. Load `.spi3d` and `.cube` in the fork. Both are simple text formats; a parser is a few hundred lines.
3. Upload as a `GL_TEXTURE_3D` and sample it in `UberShader` at one fixed hook.
4. Apply a log shaper before the lookup, because a 3D LUT only accepts `[0,1]` and scene-linear data is unbounded. Clamp negatives first: `log2` of a negative is NaN. A range of −12 to +8 stops covers typical scene-linear content.
5. Fold the inverse of tev's load-time matrix into the bake, so the LUT input is tev's internal Rec.709-linear and nothing in the load path changes. Bake one LUT per (source space × display view) pair.
6. Expose the choice in the UI and persist it via #3.

**Why this is exact enough.** The ACES output transform is a fixed value-to-value mapping; it does not adapt to image content. A 65³ table is the standard GPU representation and is what DCC applications ship.

**Files.** New `src/DisplayTransform.cpp` + header, a LUT parser, a hook in `src/UberShader.cpp`, a menu entry in `src/ImageViewer.cpp`.

**Verification.** Open a scene-linear AP0 image and compare against the same image rendered through the same OCIO view in a colour-managed application. Check a highlight above 1.0, a saturated primary, and a mid-grey. Confirm the pixel probe still reports unclamped source values.

**Risks.** The output-encoding trap: an OCIO display view returns *already encoded* values, while tev's framebuffer applies its own transfer. Feeding one into the other double-encodes and looks washed out. Either use a linear output view and let tev encode, or tell tev the framebuffer is sRGB. Decide deliberately and document it.

---

### 5. Folder grouping and natural sort

**Status:** specified · **Effort:** 1–2 days

**Why.** Opening a directory with `-r` tries to load every file, including JSON and Markdown, producing a wall of errors. Ordering is whatever the shell or loader produces, so `a10` sorts before `a2`. Related outputs (`foo_in`, `foo_out`, `foo_diff`) are scattered instead of adjacent, which makes flipping between a result and its reference awkward.

**Approach.**

- A `--filter` / extension allow-list, or simply skip files whose loader reports "unknown image type" without logging an error.
- Natural sort on the numeric runs in filenames.
- Optional grouping: detect a common suffix/prefix pattern and order siblings adjacently, so `Up`/`Down` cycles within a group before moving to the next.

**Files.** `src/main.cpp` (argument handling and ordering), `src/ImagesLoader.cpp`, possibly a new `src/FileOrder.cpp`.

**Verification.** A directory containing images plus `.json`/`.md` opens cleanly with no errors. `a2` precedes `a10`. A folder of `_in`/`_out`/`_diff` triplets flips in a sensible order.

**Risks.** Grouping heuristics are opinionated; make it opt-in and document the pattern syntax.

---

## Phase 2 — analysis

### 6. IPC command and query channel

**Status:** idea · **Effort:** 3–4 days

**Why.** The IPC socket can push images and vector graphics into the viewer, but cannot run commands or report state. That blocks every scripted workflow: setting a display transform, exporting, or reacting to the user flipping to a new image.

**Approach.** Extend the protocol additively, following the existing convention of new type IDs rather than changed payloads (`UpdateImageV2`, `V3`, `OpenImageV2`). Add:

- `RunCommand` — invoke a named command from the table introduced in #2.
- `QueryState` — current image name, channel group, inspection space, exposure.
- `Subscribe` — server pushes an event when the selected image changes.

Keep the framing identical: 4-byte length, 1-byte type, payload. A JSON or CBOR body for the new types is fine; the existing binary payloads stay untouched.

**Files.** `include/tev/Ipc.h`, `src/Ipc.cpp`, `src/main.cpp` (dispatch).

**Verification.** A Python script sets exposure, flips to the next image, reads back the current image name, and receives an event on selection change.

**Risks.** Protocol compatibility with existing clients. Additive type IDs are safe; changing the framing is not.

---

### 7. Live difference view

**Status:** idea · **Effort:** 2–3 days

**Why.** Comparing against a reference is visual only. Numeric differences are generated in external pipelines as separate files, which means the viewer cannot answer "how far apart are these two, right here" for an arbitrary pair.

**Approach.** tev already computes an error metric between an image and its reference for the histogram. Expose that as a displayable result: absolute difference, signed difference, or a gain-multiplied difference, shown as an extra channel group or an overlay. Keep the computation on the worker pool, not in the shader.

**Files.** `src/ImageCanvas.cpp` (metric computation), `src/Image.cpp` (`getHdrImageData` already takes a reference and metric), `src/ImageViewer.cpp` (UI).

**Verification.** Load a render and its reference, enable difference, confirm zero where identical and a plausible magnitude where not. Compare against an externally generated diff for the same pair.

**Risks.** Registration: if the two images differ in crop or size, a naive difference is meaningless. Reject mismatched sizes explicitly rather than producing garbage.

---

### 8. Per-image metrics table and CSV export

**Status:** idea · **Effort:** 3–4 days

**Why.** The histogram answers "what is the distribution" for one image at a time. It does not answer "which of these two hundred renders clips most" or "what is the mean error across the folder", which is the question that actually drives decisions.

**Approach.** Compute a small set per image: min, max, mean, standard deviation, percentage below zero, percentage above one, percentage at exactly zero (crushed blacks). Show as a sortable table, and export CSV. Reuse `CanvasStatistics` where possible so the numbers match what the histogram shows.

**Files.** `src/ImageCanvas.cpp`, new `src/StatisticsTable.cpp`, `src/ImageViewer.cpp`.

**Verification.** Export a folder and check the CSV against an independent script for three images. Confirm values update when the reference or inspection space changes.

**Risks.** Defining "clipped" is a policy decision: above 1.0 in scene-linear is not the same as above display white. State the definition in the UI.

---

### 9. Batch export through the display transform

**Status:** idea · **Effort:** 1 day, after #4

**Why.** `--convert-to` exists and applies tev's tonemap, gamma, exposure, and offset, but not a colour-managed display transform. Producing a contact sheet or a figure set through the same view you inspect in currently requires another application.

**Approach.** Once #4 exists, let `--convert-to` use the selected display transform. Add a flag to name the LUT or view explicitly so exports are reproducible from a command line.

**Files.** `src/main.cpp` (`convertTo`), `src/DisplayTransform.cpp`.

**Verification.** Export the same image through the viewer's view and through the CLI; the files should be identical.

---

### 10. Measurement and plot overlays from scripts

**Status:** idea · **Effort:** client only, no C++

**Why.** A plot of a metric against image index, or a region annotation, is often more useful on top of the image than in a separate window.

**Approach.** Use the existing `VectorGraphics` IPC packet. It carries NanoVG commands — paths, rectangles, arcs, text, colours, stroke widths — stored per image and rendered over both the current image and the reference. A plot is polylines plus tick marks plus text labels.

Write a small Python client (about 150 lines: 4-byte length prefix, 1-byte type, little-endian payload) and a plot helper that maps data coordinates into image pixel coordinates. Send `append=false` to replace the overlay on each update.

**Limitations to design around.** Coordinates are image space, so overlays zoom and pan with the image. Text uses one font, with no rotation. There is no axes helper; ticks and labels are drawn explicitly.

**Verification.** A script reads a folder, computes one number per image, and draws a sparkline in the corner of each image. Flipping shows the correct plot per image.

---

## Phase 3 — scale

### 11. Streaming / prefetch cache

**Status:** deferred · **Effort:** 2–3 weeks

**Why.** Every opened image is decoded and held in RAM. Measured: 122 images totalling 2.6 GB occupy about 7.9 GB resident. A folder of thousands of images is not openable.

**Approach.** Introduce an eviction policy and a prefetch window: keep the current image plus N neighbours decoded, drop the rest, reload on demand. This is invasive — `Image` currently owns its converted channels for the lifetime of the session, and the canvas assumes they are resident. It also interacts with reference comparison, which needs two images at once.

**Why deferred.** It is the largest single change in this document and it only matters once the working set exceeds RAM. Until then, opening a subset is cheaper than building a cache.

---

### 12. Ratings, tags, and filtering

**Status:** deferred · **Effort:** ~1 week

**Why.** Triaging thousands of images needs keep/reject marks and a way to filter by them. A viewer without ratings forces external bookkeeping.

**Approach.** Sidecar file per folder (JSON) mapping filename to rating, tags, and a note. Filter the image strip by rating. Keep it out of the image files themselves.

**Why deferred.** This is catalogue behaviour, not analysis. It is also the point where the tool starts competing with asset managers, which is a different product.

---

## Phase 4 — deferred, product decisions

### 13. Full OpenColorIO linkage

**Status:** deferred · **Effort:** 2–4 weeks

**Why.** A LUT covers a fixed display view. Linking OpenColorIO would add config browsing, arbitrary looks, and dynamic parameters, at the cost of a dependency chain (yaml-cpp, expat, pystring, minizip), shader generation, and permanent merge conflicts in the shader.

**Approach.** OpenColorIO's GPU renderer emits GLSL for an arbitrary transform. Integrate it by prepending the generated functions to `UberShader` and calling them at the same hook that #4 uses. The seam is identical, which is why #4 is not throwaway work.

**Decision point.** Only worth it if the viewer becomes the reference monitor for colour sign-off, rather than a fast inspector.

---

## Non-goals

- **Compositing.** No layers, no nodes, no roto. Use a compositor.
- **Video playback.** Folder flipping is the requirement; timelines are not.
- **Asset management.** Ratings are the limit; no database, no thumbnails, no ingest.
- **Publication plotting inside the viewer.** The viewer shows live diagnostics; publication figures belong in a plotting library.
- **A plugin ABI.** Dynamic plugins cost months and create a permanent compatibility burden. Static, in-tree extension is the policy.

## Open questions

1. Should the display transform be per-image (from metadata) or global (from a menu)? Per-image is correct; global is predictable. Probably both, with an override.
2. Is the inspection colour space a property of the document or of the session? Persisting it (#3) implies the session, which is probably right for a scientific tool.
3. How should a folder of mixed source spaces be handled when a single display transform is selected? One LUT per source space, chosen per image, is the honest answer.
4. Do we upstream any of this? `colorInteropID` and the arrow-key option are small and generic; the rest is opinionated. Upstreaming reduces maintenance even if the rest stays here.

## Handoff checklist

Before starting a feature:

- [ ] `git fetch upstream && git log --oneline main..upstream/master` — know what upstream has changed.
- [ ] Read `FORK.md` rules; check whether the feature can live in a new file.
- [ ] Confirm the build is clean before changing anything: `cmake --build build -j`.

Before calling a feature done:

- [ ] Verified against real data, not a synthetic test image alone.
- [ ] `git diff` contains no reformatting of upstream files.
- [ ] Commit message states the reason, not just the change.
- [ ] This document updated: status, and anything learned that the next person needs.
