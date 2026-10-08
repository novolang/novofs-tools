# novofs-tools

novofs-tools is the host's side of
[novofs](https://novo-lang.org/packages/novofs), the power-cut-safe
filesystem for a microcontroller's flash.  It holds an image tool that
builds a volume from a folder for a board's data region and reads one
back, a flash device in the host's memory with power-cut injection for
testing a program that keeps a volume, a device over an image file,
an adapter from embedded-hal-nv's `BlockDevice`, and whole files and
listings over any novofs `FileSystem`.

## What it is

A novofs volume is a set of erase blocks with metadata pairs and data
blocks in them, as novofs's README describes.  A volume image is the
same bytes in a file: block `b` of a volume of blocks of `block_size`
bytes is the `block_size` bytes at `b * block_size` of the file.  A
board's data region holds such an image at the region's first byte, so
an image built on the host for the board's geometry is the volume the
board mounts, and the region read back through a probe is an image the
host reads.

The image tool builds an image from a folder, lists and prints its
files, checks the whole tree, reports its block use, and unpacks it to
a folder.  The geometry of a board's data region comes from the
toolchain, `novo bsp region --target=<board>`, so a build for a board
types no geometry.

A test of a program that keeps a volume needs a device that behaves as
NOR flash and fails as flash fails.  `RamFlash` keeps its bytes in the
host's memory, refuses a program or an erase that breaks NOR rules, and
cuts power after a chosen number of programs and erases, optionally
tearing the operation it cuts.

## Install

```sh
novo install novofs-tools          # the image tool, into ~/.novo/bin
novo pkg add novofs-tools          # the devices and the wrappers, in a host program
```

## Example

A folder becomes the volume of a board's data region, and the probe
writes it there:

```sh
mkdir -p files/etc
printf 'hello from the host\n' > files/readme.txt
printf 'rate=100\n' > files/etc/config
novofs-tools mk files -o files.bin --target=nrf52832-dk
novo flash --target=nrf52832-dk --data files.bin
```

```text
mk: files.bin ok (5/8 blocks used), for the data region of nrf52832-dk at 0x78000, 8 blocks of 4096 bytes
novo: writing files.bin into the board nrf52832-dk's data region (block_region), 0x78000 to 0x80000, 8 blocks of 4096 bytes (chip=nRF52832_xxAA, probe=1366:1015:000682139029)
novo: the data region holds files.bin, read back and compared; the board was reset
```

A host test of a program's storage, on a RAM device whose power is cut
in the middle of the third operation:

```novo norun:needs-pkg
use std.bytes
use nvfs
use hostfs
use ramflash

fn main() [io, mutate]
    // 32 blocks of 512 bytes, read and programmed in 16-byte pages
    let dev = ramflash.ram_flash(16, 16, 512, 32)
    let _ = nvfs.format(dev, nvfs.default_config())
    let vol = nvfs.mount(dev)
    let _ = hostfs.file_write(vol, "/log", bytes.from_str("one"))
    // The third prog or erase from here is cut, half applied.
    ramflash.ram_schedule_cut(3, 8)
    let _ = hostfs.file_write(vol, "/log", bytes.from_str("two"))
    ramflash.ram_clear_cut()
    // Power back: the volume holds the file as it was or as it is now.
    let again = nvfs.mount(vol.unmount())
    match hostfs.file_read(again, "/log")
        FOk(b)  => println(bytes.to_str(b))      // "one" or "two"
        FErr(e) => println(nvfs.err_str(e))
```

## What the package contains

| Module | What is in it |
|---|---|
| `main` | The image tool: `mk`, `ls`, `cat`, `fsck`, `df`, `extract` and `selfcheck` |
| `ramflash` | `RamFlash`, a flash device in the host's memory with NOR rules, erase counts, failing blocks and power-cut injection |
| `fileflash` | `FileFlash`, a volume image in a file as a flash device |
| `blockflash` | `BlockFlash`, any embedded-hal-nv `BlockDevice` as a flash device, and `RamBlock`, a block device in memory |
| `hostfs` | Whole files as `Bytes` and directories as lists over any `FileSystem`, and a device's bytes read and programmed directly |

## How to choose an entry point

- Preparing a volume for a board, or reading one copied off a board:
  the image tool.
- Testing a program's storage on the host: `RamFlash`, with
  `ram_schedule_cut` for a power cut and `ram_mark_bad` for a failing
  block.
- Reading or writing an image file from a program: `FileFlash`.
- A volume on an SD card or another whole-block device: `BlockFlash`
  over its `BlockDevice`.
- Whole files and listings from a program on the host: `hostfs`, over
  a novofs volume or another filesystem that implements `FileSystem`.

## The image tool

```bash
novofs-tools mk ./tree -o image.bin --block-size 4096 --block-count 256
novofs-tools mk ./tree -o image.bin --target=nrf52832-dk   # the board's data region
novofs-tools ls image.bin                 # kinds and sizes
novofs-tools ls image.bin /sub
novofs-tools cat image.bin /readme.txt    # text output; extract is byte-exact
novofs-tools fsck image.bin               # reads the whole tree and counts it
novofs-tools df image.bin                 # block use
novofs-tools extract image.bin -o ./out   # a byte-exact copy of the tree
```

## The rules a user needs

1. `mk` is deterministic: the same folder and flags give a byte-identical
   image, from a sorted walk and an allocation that depends only on the
   volume's cursor.
2. `mk` must be told the target's geometry, typed or with
   `--target=<board>`, because the prog size shapes the metadata
   layout.  The read verbs find the superblock in block 0, or in block
   1 when block 0 was caught in a compaction, and need no flags.
3. A geometry option typed beside `--target` must agree with the
   board's, or `mk` refuses it naming both.  A folder that does not fit
   is refused with the shortfall in blocks and bytes, and no image is
   written.
4. `mk --target` asks the toolchain at `$NOVO`, else `novo` on PATH,
   for the board's region.  `novo flash --data` refuses an image whose
   size, block size, block count or page is not the board's.
5. Every error exits nonzero.
6. A RamFlash's erase counts, failing blocks and power-cut schedule are
   module state and describe the most recently made RamFlash.  Two
   RamFlash values over the same cells see each other's writes.

## What is not included

- A device for a board: novofs's `RegionFlash` is the board's.
- An image of a littlefs volume: novofs's format is its own.

## Related packages

- [novofs](https://novo-lang.org/packages/novofs) is the filesystem a
  program on a board mounts, and the `Flash` and `FileSystem` traits
  these devices and wrappers are written against.
- [embedded-hal-nv](https://novo-lang.org/packages/embedded-hal-nv)
  defines the `BlockDevice` trait `BlockFlash` adapts.
- [crc-nv](https://novo-lang.org/packages/crc-nv) computes the CRC-32C
  the image tool checks a superblock record with.

## Tests

`novo test tests` runs `tests/host_tests.nv`: RamFlash's NOR rules,
failing blocks and power cuts; FileFlash over an image file, on a full
device and past the end of an image; whole files and listings over a
novofs volume on both devices and over a filesystem of the test's own
whose reads fail; and RamBlock and the block-device adapter.  The
novo-lang repository runs the image tool's checks: every verb, the
determinism of `mk`, a board's geometry, the refusals, and an image
written by novofs 0.1.0 read and written again.

## Licence

Apache-2.0.  See [LICENSE](LICENSE).
