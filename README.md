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
| [ASUS Tinker Board 3](tinker-board-3/) | Rockchip RK3566 | aarch64 | ⚠️ On hold — ethernet DMA | [boot-chain.md](tinker-board-3/boot-chain.md) |

## Boot Stage Comparison

| Stage | Milk-V Mars | Orange Pi RV | Tinker Board 3 |
|-------|-------------|--------------|----------------|
| **Boot ROM** | JH7110 — finds SPL by partition type GUID | JH7110 — same | RK3566 — raw offset at sector 64 |
| **SPL** | U-Boot SPL v2026.01, SPI NOR | U-Boot SPL v2026.01, SPI NOR | idbloader (DDR init + SPL), SD raw |
| **Secure firmware** | OpenSBI v1.8.1 (in U-Boot FIT) | OpenSBI v1.8.1 (in U-Boot FIT) | TF-A BL31 (in idbloader/FIT) |
| **Bootloader** | U-Boot v2026.01, SPI NOR | U-Boot v2026.01, SPI NOR | U-Boot v2024.x, SD raw |
| **Boot manager** | systemd-boot (bootctl installed) | systemd-boot (fallback path) | systemd-boot (bootctl installed) |
| **Kernel** | Linux 6.19.0, mainline DTS | Linux 6.19.0, custom DTS | Linux 7.0-rc4, custom DTS (⚠️ ethernet hold) |
| **initrd** | None — no-initrd build | None — no-initrd build | dracut initramfs ✅ |
| **root= cmdline** | Block device path required | Block device path required | UUID safe (initrd resolves it) |
| **Rootfs device** | SD card | NVMe | SD card |
| **efivarfs** | rw (EFI_RT_VOLATILE_STORE=y) | ro | rw |
| **OS** | Fedora 43 riscv64 | Fedora 43 riscv64 | Fedora 43 aarch64 |

## Architecture Comparison

| | RISC-V (JH7110) | AArch64 (RK3566) |
|---|---|---|
| Secure firmware | OpenSBI (RISC-V SBI spec) | TF-A BL31 (ARM TrustZone) |
| Boot ROM discovery | Partition type GUID | Raw sector offset |
| initrd | Not used — drivers must be =y | dracut initramfs standard |
| `root=` cmdline | Block device path only | UUID/LABEL safe |
| `dnf update kernel` | Manual (no kernel-install DTB plugin) | Automatic via kernel-install + BLS |

## JH7110 Gotchas (riscv64 boards)

- **Partition type GUIDs** — Boot ROM finds bootloader partitions by type GUID,
  not partition number. Wrong GUIDs = silent failure, board appears dead.
  SPL GUID: `2E54B353-1271-4842-806F-E436D6AF6985`
  U-Boot FIT GUID: `BC13C2FF-59E6-4262-A352-B275FD6F7172`

- **No initrd** — storage drivers must be built-in (`=y`). Any driver in the
  path between power-on and rootfs mount must be compiled in.

- **EFI variable store** — U-Boot uses `ubootefi.var` on the ESP.
  `CONFIG_EFI_RT_VOLATILE_STORE=y` required for writable efivarfs.

- **SPI NOR reflash** — use U-Boot `sf update`, never `dd` to `/dev/mtdX`.
  NOR flash requires erase before write — `dd` corrupts silently.

- **root= cmdline** — explicit device path required, not UUID.

## RK3566 Gotchas (aarch64 boards)

- **idbloader offset** — written to sector 64 (outside partition table), not
  to a partition. Standard `dd` with `seek=64` or `rkdeveloptool`.

- **Kernel/DTB pairing** — BSP kernel requires BSP DTB; mainline kernel requires
  mainline DTB. Never mix them. The panic you get is not informative.

## Build Environment

Builds use mainline sources:
- **riscv64:** RISC-V builder VM (x86_64 host, riscv64 cross-compiler), U-Boot v2026.01, OpenSBI v1.8.1, Linux v6.19
- **aarch64:** AArch64 builder VM (native aarch64), Linux v6.19 / v7.0-rc4
- Fedora 43 packages from [riscv-kojipkgs.fedoraproject.org](https://riscv-kojipkgs.fedoraproject.org/repos/f43-build/latest/riscv64/) (riscv64) and standard Fedora mirrors (aarch64)
