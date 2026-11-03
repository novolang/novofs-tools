# Changelog

All notable changes to novofs-tools are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.2 — 2026-11-03

This release requires novo 0.23.0 or later.

## 0.1.1 — 2026-10-25

The package needs Novo 0.21.1 or later and depends on novofs 0.3.1 (the release that fixes two data-loss bugs after a failed device read).  Calls that pass a variable to a `var` parameter now write the `var` marker (E2050): the block adapter's `read_block` and `write_block` calls, and `dir_list`'s listing cursor, which is typed `Listing`.  The package now carries tests for the RAM flash, the block adapter, the image-file device and the whole-file helpers, and measures 100% line coverage.  No signature or behaviour changed.

## 0.1.0 — 2026-10-08

The host's side of novofs, which novofs 0.2.1 carried in its own
package.  It depends on novofs 0.3.0.

- The image tool, installed as `novofs-tools`: `mk` (with
  `--target=<board>`), `ls`, `cat`, `fsck`, `df`, `extract` and
  `selfcheck`, as novofs 0.2.1's `novofs` command had them.  `fsck` and
  `extract` walk any `FileSystem`.
- `ramflash`: `RamFlash` and its erase counts, failing blocks and power
  cuts.  The cells are a list, so the device is `Flash[mutate]` where
  novofs 0.2.1's was `Flash[mutate, ffi]`.
- `fileflash`: `FileFlash`, a volume image in a file.
- `blockflash`: `BlockFlash` over embedded-hal-nv's `BlockDevice`, now
  `Flash[hw, mutate]`, and `RamBlock`, a block device in memory.
- `hostfs`: `dir_list`, `file_read`, `file_read_at`, `file_write`,
  `file_append`, `src`, `prog_bytes` and `read_bytes`, generic over any
  `FileSystem` rather than a novofs volume.  A `DirEntry` carries the
  entry's name, whether it is a directory and its size; an inline
  file's bytes are read with `file_read`.
