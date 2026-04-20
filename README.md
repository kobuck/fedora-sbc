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
- **Fedora packages as the default** — standard Fedora package repositories and
  userspace wherever available; custom builds only where Fedora packaging gaps
  require them, with the gap explicitly identified

The goal in each case is a board that boots to a functional Fedora userspace and
can receive kernel and package updates through normal Fedora tooling.

## Scope

These examples cover the firmware and boot chain layer. The kernel strategy
varies by architecture, driven by Fedora packaging availability:

**AArch64 boards** use the Fedora-packaged kernel installed via `dnf` where the
board's DTB is included in the Fedora aarch64 kernel package. dracut generates an
initramfs automatically via kernel-install, and `dnf update kernel` works without
manual intervention. UUID references in boot entries are safe. Where the Fedora
aarch64 kernel package does not yet include a board's DTB (e.g. Allwinner H618),
a custom kernel build is required — the same packaging-gap pattern as riscv64.

**RISC-V (riscv64) boards** have no kernel RPM in standard Fedora repos —
riscv64 builds exist on Fedora's Koji infrastructure but cannot be installed
cleanly alongside a standard Fedora system without repo conflicts. A custom-built
kernel is therefore required. To avoid needing an initrd for early driver loading,
all drivers in the rootfs path are compiled in (`=y`). This is a workaround for
a packaging gap, not a design preference — when Fedora ships riscv64 kernel RPMs
in standard repos, these boards should migrate to the packaged kernel + dracut path.

The examples show where the goal was reached, where it was partially reached,
and where it is blocked. A working result and an honest status report on an
incomplete one are both useful outputs.

## Boards

| Board | SoC | Arch | Status | Boot Chain |
|-------|-----|------|--------|------------|
| [Radxa Rock Pi 4 Plus](rock-pi-4-plus/) | Rockchip RK3399 | aarch64 | ✅ Fedora 43 | [boot-chain.md](rock-pi-4-plus/boot-chain.md) |
| [Milk-V Mars](milkv-mars/) | StarFive JH7110 | riscv64 | ✅ Fedora 43 | [boot-chain.md](milkv-mars/boot-chain.md) |
| [Orange Pi RV](orangepi-rv/) | StarFive JH7110 | riscv64 | ✅ Fedora 43 | [boot-chain.md](orangepi-rv/boot-chain.md) |
| [ASUS Tinker Board 3](tinker-board-3/) | Rockchip RK3566 | aarch64 | ⚠️ On hold — ethernet DMA | [boot-chain.md](tinker-board-3/boot-chain.md) |
| [Raspberry Pi 3 Model B](raspberry-pi-3b/) | Broadcom BCM2837 | aarch64 | ✅ Fedora 43 | [boot-chain.md](raspberry-pi-3b/boot-chain.md) |
| [Orange Pi Zero 3](orangepi-zero3/) | Allwinner H618 | aarch64 | ✅ Fedora 43 | [boot-chain.md](orangepi-zero3/boot-chain.md) |

## Boot Stage Comparison

| Stage | Rock Pi 4 Plus | Milk-V Mars | Orange Pi RV | Tinker Board 3 | Raspberry Pi 3B | Orange Pi Zero 3 |
|-------|----------------|-------------|--------------|----------------|-----------------|
| **Boot ROM** | RK3399 — raw offset at sector 64 | JH7110 — partition type GUID | JH7110 — partition type GUID | RK3566 — raw offset at sector 64 | BCM2837 — VideoCore GPU ROM, FAT files | H618 (sunxi) — raw offset at sector 16; GPT entry array relocated to sector 2210 |
| **SPL** | idbloader (DDR init + SPL), SD raw | U-Boot SPL v2026.01, SPI NOR | U-Boot SPL v2026.01, SPI NOR | idbloader (DDR init + SPL), SD raw | bootcode.bin + start.elf (GPU firmware) | U-Boot SPL v2026.04, SD card sector 16 (raw) |
| **Secure firmware** | TF-A BL31 v2.12.0 (in U-Boot FIT) | OpenSBI v1.8.1 (in U-Boot FIT) | OpenSBI v1.8.1 (in U-Boot FIT) | TF-A BL31 (in idbloader/FIT) | None (GPU handles ARM release) | TF-A BL31 v2.12.0 (in U-Boot FIT) |
| **Bootloader** | U-Boot v2026.01, SD raw | U-Boot v2026.01, SPI NOR | U-Boot v2026.01, SPI NOR | U-Boot v2024.x, SD raw | U-Boot v2026.01, FAT file (loaded as "kernel") | U-Boot v2026.04, SD card raw |
| **Boot manager** | systemd-boot (bootctl installed) | systemd-boot (bootctl installed) | systemd-boot (fallback path) | systemd-boot (bootctl installed) | systemd-boot (fallback path) | systemd-boot (fallback path — efivarfs ro) |
| **Kernel** | Linux 6.19.9, Fedora-packaged | Linux 6.19.0, custom build (no riscv64 RPM in Fedora repos) | Linux 6.19.0, custom build (no riscv64 RPM in Fedora repos) | Linux 7.0-rc4, Fedora-packaged (⚠️ ethernet hold) | Linux 6.19.11, Fedora-packaged | Linux 7.0.0-rc6, custom build (no H618 DTB in Fedora aarch64 kernel package) |
| **initrd** | dracut initramfs (via kernel-install) | None — custom build, rootfs-path drivers =y | None — custom build, rootfs-path drivers =y | dracut initramfs (via kernel-install) | dracut initramfs (via kernel-install) | dracut initramfs (QEMU aarch64 chroot on builder) |
| **root= cmdline** | UUID safe (initrd resolves it) | Block device path required | Block device path required | UUID safe (initrd resolves it) | UUID safe (initrd resolves it) | Block device path (`/dev/mmcblk0p2`) |
| **Rootfs device** | SD card | SD card | NVMe | SD card | SD card | SD card |
| **efivarfs** | rw (EFI_RT_VOLATILE_STORE=y) | rw (EFI_RT_VOLATILE_STORE=y) | ro | rw | rw (EFI_RT_VOLATILE_STORE=y) | ro (EFI_RT_VOLATILE_STORE not set) |
| **OS** | Fedora 43 aarch64 | Fedora 43 riscv64 | Fedora 43 riscv64 | Fedora 43 aarch64 | Fedora 43 aarch64 | Fedora 43 aarch64 |

## Architecture Comparison

| | RISC-V (JH7110) | AArch64 (RK3399 / RK3566) | AArch64 (BCM2837) | AArch64 (H618) |
|---|---|---|---|
| Secure firmware | OpenSBI (RISC-V SBI spec) | TF-A BL31 (ARM TrustZone) | None — GPU handles ARM core release | TF-A BL31 (ARM TrustZone) |
| Boot ROM discovery | Partition type GUID | Raw sector offset (sector 64) | FAT files on MBR-visible partition | Raw sector offset (sector 16); GPT entry array relocated |
| Partition table | GPT | GPT | GPT + hybrid MBR (see BCM2837 notes) | GPT (entry array relocated via `sgdisk -j`) |
| Kernel source | Custom build — no riscv64 RPM in standard Fedora repos | Fedora-packaged (`dnf install kernel`) | Fedora-packaged (`dnf install kernel kernel-modules`) | Custom build — no H618 DTB in Fedora aarch64 kernel package |
| initrd | None — custom build, rootfs-path drivers =y | dracut initramfs (via kernel-install) | dracut initramfs (via kernel-install) | dracut initramfs (QEMU aarch64 chroot) |
| `root=` cmdline | Block device path only | UUID/LABEL safe | UUID/LABEL safe | Block device path (UUID deferred) |
| `dnf update kernel` | Manual (no kernel-install DTB plugin yet) | Automatic via kernel-install + BLS | Automatic via kernel-install + BLS | Manual (custom kernel, no RPM) |

## JH7110 Notes (riscv64 boards)

- **Partition type GUIDs** — The Boot ROM locates bootloader partitions by type GUID,
  not partition number. Incorrect GUIDs produce a silent failure — no error output,
  the board appears dead.
  SPL GUID: `2E54B353-1271-4842-806F-E436D6AF6985`
  U-Boot FIT GUID: `BC13C2FF-59E6-4262-A352-B275FD6F7172`

- **Custom kernel (packaging gap)** — Fedora does not ship riscv64 kernel RPMs in
  standard repos. A custom-built kernel is required. All drivers in the rootfs path
  must be built-in (`=y`) — without an initrd there is no early userspace to load
  modules before rootfs mount.

- **EFI variable store** — U-Boot uses `ubootefi.var` on the ESP.
  `CONFIG_EFI_RT_VOLATILE_STORE=y` required for writable efivarfs.

- **SPI NOR reflash** — Use U-Boot `sf update`, never `dd` to `/dev/mtdX`.
  NOR flash requires erase before write; `dd` corrupts silently.

- **root= cmdline** — Explicit device path required, not UUID.

## RK3399 / RK3566 Notes (aarch64 boards)

- **idbloader offset** — Written to sector 64 (raw, outside partition table).
  Partitions must start at sector 32768 or later to avoid overlap.
  `sgdisk --zap-all` overwrites sector 64 — always reflash U-Boot after
  repartitioning.

- **TF-A build (RK3399)** — Requires a Cortex-M0 cross-compiler for the PMU
  firmware. Fedora ships `arm-linux-gnu-gcc`; pass `M0_CROSS_COMPILE=arm-linux-gnu-`
  to the TF-A build (the default `arm-none-eabi-gcc` name is not used on Fedora).

- **Kernel/DTB pairing** — BSP kernel requires BSP DTB; mainline kernel requires
  mainline DTB. Mixing them produces an uninformative panic.

## H618 Notes (Orange Pi Zero 3)

- **GPT + sunxi SPL conflict** — The H618 Boot ROM hardcodes the SPL at byte
  8192 (sector 16), which falls inside the standard GPT partition entry array
  (sectors 2–33). Writing U-Boot destroys the primary GPT entry array on every
  flash. Solution: `sgdisk -j 2210` relocates the entry array to sector 2210,
  past the end of the ~1.1MB U-Boot binary. The backup GPT is unaffected; the
  primary GPT survives all subsequent U-Boot writes. This is distinct from the
  RK3399/RK3566 approach (sector 64, partition start ≥32768) — each SoC family
  has its own raw offset to check against the actual binary size.

- **U-Boot SPL config overrides** — Three values are absent from
  `orangepi_zero3_defconfig` and must be set explicitly after `make defconfig`:
  ```
  CONFIG_SPL_SYS_MALLOC_F_LEN=0x10000
  CONFIG_SPL_STACK_R_MALLOC_SIMPLE_LEN=0x200000
  CONFIG_SYS_MMCSD_RAW_MODE_U_BOOT_SECTOR=0x50
  ```
  Without these, SPL fails with "alloc space exhausted" / "Could not get FIT
  buffer". Verify the FIT sector offset by scanning the binary for FIT magic
  bytes (`d00dfeed`).

- **Custom kernel (packaging gap)** — The H618 DTS is in mainline from kernel
  6.1 onward, but the Fedora aarch64 kernel package does not yet include the
  H618 DTB. A custom kernel build is required. Unlike riscv64, aarch64 supports
  dracut normally — the initramfs is built in a QEMU aarch64 chroot on the
  builder VM.

- **zram units** — Mask `systemd-zram-setup@zram0.service` and
  `zram-generator.service`. Masking `dev-zram0.device` alone is insufficient —
  the generator instantiates the setup service directly under
  `/run/systemd/generator/` at boot.

## BCM2837 Notes (Raspberry Pi 3B)

- **Partition table: GPT + hybrid MBR** — The BCM2837 GPU ROM reads sector 0 as
  an MBR partition table to find the FAT boot partition. Pure GPT produces a
  silently non-booting card. The solution is a hybrid MBR: GPT is the primary
  partition table, with a legacy MBR partition 1 entry (type `0x0C`) overlaid
  using `gdisk`'s hybrid MBR feature (`r` → `h`). The GPU ROM finds the FAT
  partition via the MBR entry; U-Boot and everything above sees clean GPT.
  RPi 4/5 have GPT-capable EEPROM firmware and do not require this.

- **GPU firmware is the boot ROM** — The BCM2837 boots from a VideoCore GPU ROM,
  not the ARM CPU. The GPU executes `bootcode.bin` → `start.elf` → loads U-Boot
  as the "kernel" binary, then releases the ARM cores. This is fundamentally
  different from every other board here — there is no idbloader, no SPL, no SPI
  NOR, and no way to bypass the GPU firmware stage.

- **kernel-modules required** — Install both `kernel-core` and `kernel-modules`.
  `kernel-core` alone omits BCM2835 peripheral drivers. The dracut initramfs
  must explicitly include: `bcm2835-dma bcm2835 mmc_core mmc_block`.

- **GPU firmware modifies the DTB** — `start.elf` applies device tree overlays
  before passing the DTB to U-Boot. `dtoverlay=disable-bt` in `config.txt` is
  required to correctly configure the sdhost controller in 64-bit mode.

- **Mini-UART serial console** — `ttyS0` at `3f215040`, not the PL011 (which is
  wired to Bluetooth). `core_freq=250` in `config.txt` required for stable baud rate.

## Build Environment

Builds use mainline sources:
- **riscv64:** RISC-V builder VM (x86_64 host, riscv64 cross-compiler), U-Boot v2026.01, OpenSBI v1.8.1, Linux v6.19
- **aarch64 (Rockchip):** AArch64 builder VM (x86_64 host, aarch64-linux-gnu- cross-compiler), U-Boot v2026.01, TF-A v2.12.0
- **aarch64 (BCM2837):** Fedora-packaged kernel and U-Boot v2026.01; no custom kernel build required
- Fedora 43 packages from [riscv-kojipkgs.fedoraproject.org](https://riscv-kojipkgs.fedoraproject.org/repos/f43-build/latest/riscv64/) (riscv64) and standard Fedora mirrors (aarch64)
