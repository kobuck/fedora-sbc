# fedora-sbc

Fedora on single-board computers — annotated boot chains, kernel configs,
and build notes from working systems.

These are reference documents for people building Fedora on SBC hardware,
not tutorials or maintained software. The goal is that someone working on
the same or similar hardware finds useful content rather than vendor
documentation and silence.

## Boards

| Board | SoC | Arch | Status | Boot Chain |
|-------|-----|------|--------|------------|
| [Milk-V Mars](milkv-mars/) | StarFive JH7110 | riscv64 | ✅ Fedora 43 | [boot-chain.md](milkv-mars/boot-chain.md) |
| [Orange Pi RV](orangepi-rv/) | StarFive JH7110 | riscv64 | ✅ Fedora 43 | [boot-chain.md](orangepi-rv/boot-chain.md) |

## Boot Stage Comparison

| Stage | Milk-V Mars | Orange Pi RV |
|-------|-------------|--------------|
| **Boot ROM** | JH7110 — finds SPL by partition type GUID | JH7110 — same |
| **SPL** | U-Boot SPL v2026.01, SPI NOR mtd0 | U-Boot SPL v2026.01, SPI NOR mtd0 |
| **Firmware** | OpenSBI v1.8.1 (in U-Boot FIT) | OpenSBI v1.8.1 (in U-Boot FIT) |
| **Bootloader** | U-Boot v2026.01, SPI NOR mtd2 | U-Boot v2026.01, SPI NOR mtd2 |
| **Boot manager** | systemd-boot 258.7 (fully installed via bootctl) | systemd-boot (fallback path only) |
| **Kernel** | Linux 6.19.0, mainline Mars DTS | Linux 6.19.0, custom DTS |
| **Rootfs device** | SD card (mmcblk1p4) | NVMe (nvme0n1p2) |
| **Critical built-in drivers** | MMC_DW_STARFIVE | PCIe + NVMe stack |
| **Board match** | EEPROM prefix `MARS` — native mainline | EEPROM `VF7110B1` + DTS patch |
| **efivarfs** | rw (`EFI_RT_VOLATILE_STORE=y`) | ro (option not set) |
| **OS** | Fedora 43 riscv64 | Fedora 43 riscv64 |

## JH7110 Gotchas (applies to both boards)

- **Partition type GUIDs** — the Boot ROM locates bootloader partitions by type GUID,
  not partition number. Wrong GUIDs = silent failure, board appears dead with no output.
  SPL GUID: `2E54B353-1271-4842-806F-E436D6AF6985`
  U-Boot FIT GUID: `BC13C2FF-59E6-4262-A352-B275FD6F7172`

- **No initrd** — storage drivers must be built-in (`=y`), not modules (`=m`).
  Any driver in the path between power-on and rootfs mount must be compiled in.
  Wrong setting = silent boot hang, easy to mistake for hardware failure.

- **EFI variable store** — U-Boot uses a file on the ESP (`ubootefi.var`) rather
  than hardware NVRAM. `CONFIG_EFI_RT_VOLATILE_STORE=y` is required for writable
  efivarfs. Without it, `bootctl install` fails silently.

- **SPI NOR reflash** — use U-Boot `sf update` commands, never `dd` to `/dev/mtdX`.
  NOR flash requires an erase cycle before writing — `dd` skips it and corrupts silently.
  Recovery: boot from SD card, reflash via `sf update`.

- **root= in kernel cmdline** — use explicit device path (`root=/dev/mmcblk1p4`),
  not UUID. Without initrd there is no early userspace to resolve UUID references.

## What's Here

Each board directory contains:
- **boot-chain.md** — annotated description of every boot stage from power-on
  to running OS, explaining what each stage does and why it matters

## Build Environment

Builds use mainline sources on a RISC-V builder VM (x86_64 host, riscv64
cross-compiler):
- U-Boot v2026.01
- OpenSBI v1.8.1
- Linux v6.19
- Fedora 43 riscv64 ([riscv-kojipkgs.fedoraproject.org](https://riscv-kojipkgs.fedoraproject.org/repos/f43-build/latest/riscv64/))
