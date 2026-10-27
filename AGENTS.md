# AGENTS.md

Guidance for AI agents working in this repository. Keep it short; the detail lives in the referenced documents.

## What this is

Lumina Inspector: a soft fork of [tev](https://github.com/Tom94/tev), presented as a scientific and technical image analysis tool for HDR and floating-point data. GPL-3.0, public repository.

Read before changing code:

- `FORK.md` — branch model, upstream merge procedure, build and install, change rules.
- `docs/features.md` — roadmap, per-feature approach and verification.
- `docs/codebase-notes.md` — where things live in tev, the extension seams, and the traps.

## Non-negotiable rules

1. Soft fork. Upstream is the source of truth for everything we have not touched. Never rewrite published history.
2. New behaviour goes in new files. Touch upstream files only at registration seams.
3. No reformatting, renaming, or drive-by cleanup of upstream code. This is the largest source of merge pain.
4. One concern per commit, and `main` must build at every commit.
5. Do not rename the C++ namespace `tev` or the CMake project. The product is Lumina Inspector; the internals stay upstream's.
6. `LICENSE.txt` stays verbatim. Attribution belongs in `NOTICE` and the README.
7. Keep upstream copyright headers intact. Add a copyright line only to files you create.

## Build and run

```bash
cmake . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j
cmake --install build --prefix ~/.local/opt/lumina-inspector
```

Installs as `lumina-inspector`. IPC listens on `14159`; stock tev uses `14158`. Do not unify them — they must be able to run side by side.

## Verifying changes

- Test with real images, not only synthetic gradients. A scene-linear AP0 EXR, a Rec.709 EXR, and a folder of 100+ images cover most regressions.
- Check the pixel probe and histogram as well as the display. Colour bugs often appear in one and not the other.
- For UI changes, launch it. A clean build and `--version` do not prove the window works.

## Working with upstream

```bash
git fetch upstream
git log --oneline main..upstream/master
git merge upstream/master
git submodule update --init --recursive   # mandatory after every merge
```

## Style

- Conventional commit subjects; put the reason in the body.
- Prose in docs: one paragraph per line, no hard wrapping.
- Update `docs/features.md` when a feature lands, and record what the next person needs to know.

## Do not

- Commit build artifacts, `build/`, or submodule contents.
- Edit anything under `dependencies/` by hand; move the submodule pointer instead.
- Add a plugin ABI, an embedded scripting runtime, or a video timeline. See the non-goals in `docs/features.md`.
- Commit personal paths, thesis material, or credentials — the repository is public.
