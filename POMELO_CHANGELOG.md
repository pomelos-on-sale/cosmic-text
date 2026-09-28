# Pomelo changelog

What this fork changes relative to upstream `pop-os/cosmic-text`.

`pomelo-os` shapes and renders text on an ESP32-S3 (Xtensa LX7, ESP-IDF), where
the C library has no POSIX memory-mapping layer. One commit is the whole delta.

**Baseline:** upstream tag `0.15.0` (`c82ee1c5`, 2025-10-30).

**Delta:** 1 commit, 3 files, +6 / -3 -- the changelog commit that adds this
file is not counted.

## Memory-mapped font files are opt-in

`ae3e8784` · `Cargo.toml`, `src/font/mod.rs`, `src/font/system.rs`

The `std` feature pulled in `fontdb/memmap`, so every downstream that used
default features needed `mmap`, `madvise`, `mprotect` and `msync`, plus the
`MAP_*`, `PROT_*` and `MADV_*` constants. ESP-IDF's newlib has none of them,
which made the crate unbuildable on the target.

`std` no longer implies it. A new, off-by-default `fontdb-memmap` feature
enables `fontdb/memmap`, and the two `SharedFile` call sites are gated on that
feature instead: the `fontdb::Source::SharedFile` arm in `src/font/mod.rs`, and
the `make_shared_face_data` call in `src/font/system.rs`.

This costs nothing when rendering a font from memory, which is the only thing an
embedded platform does:

- `fontdb::Database::make_shared_face_data` returns early for
  `Source::Binary`, so the call site is a no-op for in-memory faces.
- mmap only ever turns a `Source::File` into a `Source::SharedFile`, and
  cosmic-text rejects that source anyway -- see the `Unsupported fontdb
  Source::File` arm in `src/font/mod.rs`, which warns and returns `None`.

So what is gone is the "memory-mapped font file" optimisation, not font support.
A desktop downstream that wants it back turns the feature on explicitly.

## Updating

```bash
git remote add upstream https://github.com/pop-os/cosmic-text.git   # once
git fetch upstream
git rebase 0.15.0        # or whatever upstream tag is being tracked
git push origin pomelo/0.15 --force-with-lease
```

Then record the new commit in `pomelo-os`:

```bash
git -C ../.. add vendor/embedded-cosmic-text
git -C ../.. commit -m "vendor: bump embedded-cosmic-text"
```

## Where this is used

`pomelo-os` consumes the fork as a path patch: `[patch.crates-io]` in its root
`Cargo.toml` points `cosmic-text` at this directory, and the submodule pins this
branch. It reaches the firmware through `iced_graphics` -- through
`embedded-iced`, which is the other fork in `vendor/`. `vendor/README.md` in
`pomelo-os` explains why the forks are excluded from its workspace.
