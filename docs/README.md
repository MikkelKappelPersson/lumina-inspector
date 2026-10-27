# Documentation

- [Feature wishlist](features.md) — what to build next, why, and how to verify it.
- [Codebase notes](codebase-notes.md) — where things live in tev, the extension seams, and the traps.

Both documents assume you have read `FORK.md` at the repository root: it defines the branch model, the rules for touching upstream code, and the build/install steps.

The short version of those rules, repeated because they matter more than anything else here:

1. New behaviour goes in new files wherever possible.
2. Upstream files are touched only at registration seams.
3. No reformatting, no renaming, no drive-by cleanups.
4. One concern per commit, and `main` stays buildable.

Every upstream release we can merge cleanly is free bug fixes and free format support. A fork that cannot merge is a fork that has to be maintained from scratch.
