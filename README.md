# fedora-sbc

Worked examples of Fedora running on SBC hardware via an open firmware stack.

These are worked examples, not a maintained project. Corrections, additions,
and reports from people working on the same or similar hardware are welcome —
open an [issue](../../issues) for specific problems or inaccuracies, or start a
[discussion](../../discussions) for questions and experience sharing.

## What These Examples Demonstrate

Each example documents a complete boot chain from power-on to a running Fedora
userspace, built on the following architectural principles:

- **Mainline U-Boot with EFI** — open-source bootloader, no vendor binary blobs
  in the boot path beyond what the SoC architecture requires (OpenSBI on RISC-V,
  TF-A on AArch64)
- **systemd-boot as the EFI boot manager** — standard UEFI handoff between
  U-Boot and the OS, Boot Loader Specification (BLS) entries for kernel management
- **Current Fedora release** — standard Fedora package repositories, standard
  userspace, no custom distribution

The goal in each case is a board that boots to a functional Fedora userspace and
can receive kernel and package updates through normal Fedora tooling.

## Scope

These examples cover the firmware and boot chain layer. They do not address:

- Building or modifying the Fedora distribution kernel for SBC hardware
- Use of dracut for initramfs-based driver loading to avoid built-in kernel config
- Integration of SBC-specific hardware beyond what is needed to reach a functional
  Fedora userspace

Those are valid approaches and valid areas of future work. They are not what
these examples demonstrate.

The examples show where the goal was reached, where it was partially reached,
and where it is blocked. A working result and an honest status report on an
incomplete one are both useful outputs.

## Boards

| Board | SoC | Arch | Status | Boot Chain |
|-------|-----|------|--------|------------|
| [Milk-V Mars](milkv-mars/) | StarFive JH7110 | riscv64 | ✅ Fedora 43 | [boot-chain.md](milkv-mars/boot-chain.md) |
| [Orange Pi RV](orangepi-rv/) | StarFive JH7110 | riscv64 | ✅ Fedora 43 | [boot-chain.md](orangepi-rv/boot-chain.md) |
| [ASUS Tinker Board 3](tinker-board-3/) | Rockchip RK3566 | aarch64 | ⚠️ On hold — ethernet DMA | [boot-chain.md](tinker-board-3/boot-chain.md) |

## Boot Stage Comparison

| Stage | Milk-V Mars | Orange Pi RV | Tinker Board 3 |
|-------|-------------|--------------|----------------|
| **Boot ROM** | JH7110 — partition type GUID | JH7110 — partition type GUID | RK3566 — raw offset at sector 64 |
| **SPL** | U-Boot SPL v2026.01, SPI NOR | U-Boot SPL v2026.01, SPI NOR | idbloader (DDR init + SPL), SD raw |
| **Secure firmware** | OpenSBI v1.8.1 (in U-Boot FIT) | OpenSBI v1.8.1 (in U-Boot FIT) | TF-A BL31 (in idbloader/FIT) |
| **Bootloader** | U-Boot v2026.01, SPI NOR | U-Boot v2026.01, SPI NOR | U-Boot v2024.x, SD raw |
| **Boot manager** | systemd-boot (bootctl installed) | systemd-boot (fallback path) | systemd-boot (bootctl installed) |
| **Kernel** | Linux 6.19.0, mainline DTS | Linux 6.19.0, custom DTS | Linux 7.0-rc4, custom DTS (⚠️ ethernet hold) |
| **initrd** | None — no-initrd build | None — no-initrd build | dracut initramfs (pre-scope, see note) |
| **root= cmdline** | Block device path required | Block device path required | UUID safe (initrd resolves it) |
| **Rootfs device** | SD card | NVMe | SD card |
| **efivarfs** | rw (EFI_RT_VOLATILE_STORE=y) | ro | rw |
| **OS** | Fedora 43 riscv64 | Fedora 43 riscv64 | Fedora 43 aarch64 |

## Architecture Comparison

| | RISC-V (JH7110) | AArch64 (RK3566) |
|---|---|---|
| Secure firmware | OpenSBI (RISC-V SBI spec) | TF-A BL31 (ARM TrustZone) |
| Boot ROM discovery | Partition type GUID | Raw sector offset |
| initrd | Not used — drivers must be =y | dracut initramfs (pre-scope approach) |
| `root=` cmdline | Block device path only | UUID/LABEL safe |
| `dnf update kernel` | Manual (no kernel-install DTB plugin) | Automatic via kernel-install + BLS |

## JH7110 Notes (riscv64 boards)

- **Partition type GUIDs** — The Boot ROM locates bootloader partitions by type GUID,
  not partition number. Incorrect GUIDs produce a silent failure — no error output,
  the board appears dead.
  SPL GUID: `2E54B353-1271-4842-806F-E436D6AF6985`
  U-Boot FIT GUID: `BC13C2FF-59E6-4262-A352-B275FD6F7172`

- **No initrd** — Storage drivers must be built-in (`=y`). Any driver in the
  path between power-on and rootfs mount must be compiled in, not loaded as a module.

- **EFI variable store** — U-Boot uses `ubootefi.var` on the ESP.
  `CONFIG_EFI_RT_VOLATILE_STORE=y` required for writable efivarfs.

- **SPI NOR reflash** — Use U-Boot `sf update`, never `dd` to `/dev/mtdX`.
  NOR flash requires erase before write; `dd` corrupts silently.

- **root= cmdline** — Explicit device path required, not UUID.

## RK3566 Notes (aarch64 boards)

- **idbloader offset** — Written to sector 64 (outside partition table), not
  to a partition. Standard `dd` with `seek=64` or `rkdeveloptool`.

- **Kernel/DTB pairing** — BSP kernel requires BSP DTB; mainline kernel requires
  mainline DTB. Mixing them produces an uninformative panic.

## Build Environment

Builds use mainline sources:
- **riscv64:** RISC-V builder VM (x86_64 host, riscv64 cross-compiler), U-Boot v2026.01, OpenSBI v1.8.1, Linux v6.19
- **aarch64:** AArch64 builder VM (x86_64 host, aarch64-linux-gnu- cross-compiler), Linux v6.19 / v7.0-rc4
- Fedora 43 packages from [riscv-kojipkgs.fedoraproject.org](https://riscv-kojipkgs.fedoraproject.org/repos/f43-build/latest/riscv64/) (riscv64) and standard Fedora mirrors (aarch64)
