# CLAUDE.md

Basic Loader is a small bootloader that can apply updates. It verifies the
firmware, installs pending firmware updates (plain, LZ4, LZ4 block-delta), and
jumps to the firmware. This checkout is a fork
(github.com/RomanMindlin/basicloader, branch `master`), used as a submodule of
`basicmac`, which is in turn a submodule of the Tabs firmware repo.

## Layout

- `src/common/` — portable code: `bootloader.h` (**public API**), `update.c`
  (update engine: plain/LZ4/delta, resumable), `lz4.c`, `sha2.c`
- `src/arm/stm32lx/` — STM32L0/L1 implementation: `bootloader.c` (main flow,
  panic and LED blink codes, flash/EEPROM writes), `boottab.h` (function table
  exported to the firmware), `startup.S`, `util.S`, linker scripts in `ld/`
  (`STM32L0xx{8,B,Z}.ld` memory sizes + `STM32L0.ld` sections)
- `src/arm/unicorn/` — simulation target used by basicmac's simulator
- `build/boards/<BOARD>/Makefile` — per-board builds (B-L072Z-LRWAN1,
  NUCLEO-L053R8, NUCLEO-L073RZ, simul-unicorn). Each sets `FLAVOR`, `MCU`, LED
  GPIOs and includes `../main.mk`
- `build/makefiles/` — `stm32lx.mk`, `unicorn.mk`, `arm.mk`, `toolchain.mk`
- `tools/fwtool/zfwtool.py` — firmware archive and update tool (click CLI);
  `signtool.py`; `tools/shavs/` SHA-256 test vectors
- `example/stm32l0/` — minimal firmware showing the header and ZFW flow
- `doc/uptypes.md` — how each update type works (with diagrams)

## Build

```sh
pip3 install -r requirements.txt     # click lz4 intelhex pycryptodome
make                                 # every board under build/boards/
make -C build/boards/B-L072Z-LRWAN1  # one board → bootloader.hex
make clean
```

Out-of-tree board builds set `TOPDIR` to this directory and include
`build/boards/main.mk`. The Tabs WSMS130 build is one: `../../bootloader/wsms130`
with `FLAVOR=stm32lx`, `MCU=STM32L071B`, and the LED on PB5. `MCU` must match
`STM32L[01]<nn>[R]<8|B|Z>`. The last letter picks the flash-size linker
script.

## How it works (STM32)

- Memory: the bootloader takes the first 12K of flash (`BLFLASH`). Firmware
  starts right after it (`FWFLASH`, 0x08003000 on the L0). The firmware begins
  with `boot_fwhdr {crc, size, entrypoint}`, and the CRC covers bytes 8..size.
- Boot: if the update pointer pair in EEPROM (`BOOT_CONFIG_BASE` =
  `DATA_EEPROM_BASE`, `fwupdate1 == fwupdate2`) points at a valid update
  (`boot_uphdr`, CRC check), the bootloader installs it. It then verifies the
  firmware CRC, clears the pointer, and returns the entrypoint. A bad CRC
  calls `boot_panic`.
- Installation is destructive but resumable. After a power loss, the next
  boot continues the update.
- The firmware calls into the bootloader through `boot_boottab`: `update`
  (validate the update and set the pointer, with an optional hash), `panic`,
  `crc32`, `wr_flash`, `sha256`. The table `version` is currently 0x108; bump
  it when you add entries.
- Panic: the bootloader blinks `BOOT_LED_GPIO`: one long flash, then type,
  reason and address. Each value is shown one nibble at a time, lowest nibble
  first, as nibble+1 blinks. The type and reason codes are in `bootloader.h`.
  `UPDATE_LED_GPIO` (optional) stays on while an update is installed.

## zfwtool

```sh
zfwtool.py create --patch [--meta k v] fw.unpatched.hex fw.zfw   # patch size/CRC, archive
zfwtool.py info fw.zfw
zfwtool.py export fw.zfw fw.hex|fw.bin
zfwtool.py mkupdate -s key.pem fw.zfw fw.up                      # LZ4 update
zfwtool.py mkupdate -d ref.zfw -s key.pem fw.zfw fw.up           # block-delta vs ref
zfwtool.py mkupdate -p ...                                       # plain
```

A delta update applies only to the exact reference firmware (`refcrc`,
`refsize`). The default block size is 4096 and must be a multiple of the flash
page size.

## Rules

- **The bootloader is never updated in the field.** `bootloader.h` and
  `boottab.h` are a frozen ABI shared with deployed firmware: don't reorder or
  resize structs, don't renumber codes, append only (keep the
  `_Static_assert`s).
- Size matters: the bootloader must fit in 12K (`-Os`). Check
  `bootloader.map` after changes.
- Code style: C gnu11, tab/4-space mix as in the existing files, a copyright
  header on every file.
- Fork changes so far: `-mthumb` in LDFLAGS (`stm32lx.mk`, `unicorn.mk`).
