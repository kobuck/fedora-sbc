## Milk-V Mars — Boot Chain with Annotations

Hardware: Milk-V Mars, StarFive JH7110 SoC (RISC-V riscv64), 8GB LPDDR4,
119GB SD card (M.2 E-Key slot unpopulated on this unit).

---

### The Boot Chain

```
JH7110 Boot ROM
  → U-Boot SPL v2026.01       (SPI NOR, mtd0, 512K partition)
  → OpenSBI v1.8.1            (embedded in U-Boot FIT)
  → U-Boot proper v2026.01    (SPI NOR, mtd2, 4M partition)
  → systemd-boot 258.7        (SD card ESP, FAT16, /boot)
  → Linux 6.19.0              (SD card ESP)
  → Fedora 43 riscv64         (SD card ext4 rootfs)
```

---

### Stage 1 — JH7110 Boot ROM

The Boot ROM is mask ROM — burned at the factory, immutable. It is the first
code that runs after power-on. Its job is to locate a valid next-stage image
and hand off.

The JH7110 Boot ROM has a configurable boot source selection controlled by two
GPIO pins (RGPIO_0 and RGPIO_1), exposed as DIP switches on the Mars. The
default factory setting is SPI Flash (both switches off), which is the
production configuration.

The Boot ROM locates bootloader partitions by **type GUID**, not by partition
number or position. The SPL partition must carry GUID
`2E54B353-1271-4842-806F-E436D6AF6985` and the U-Boot FIT partition must carry
`BC13C2FF-59E6-4262-A352-B275FD6F7172`. Using the default sgdisk type code
(`8300` Linux filesystem) causes the Boot ROM to skip the partition entirely —
no error message, the board appears dead. This is the sharpest edge in JH7110
building.

If SPI NOR is corrupted or empty, flip the DIP switch to SD card boot mode.
The Boot ROM will look for SPL on the SD card instead — the recovery path.

---

### Stage 2 — U-Boot SPL (SPI NOR, mtd0)

The SPL (Secondary Program Loader) is a minimal first-stage U-Boot that runs
from the SoC's on-chip SRAM — DRAM is not yet initialized. It initializes the
JH7110's DDR controller, brings up 8GB of LPDDR4, loads U-Boot proper from
SPI NOR into DRAM, and jumps to it.

Built from mainline U-Boot v2026.01 using `starfive_visionfive2_defconfig`.
The Mars is native in mainline U-Boot — no patches, no custom DTS required.
The SPL reads the board's EEPROM serial string (prefix `MARS`) and matches it
against a table to select the correct board DTS at runtime.

This stage produces brief output (`U-Boot SPL 2026.01`) then goes quiet while
DRAM initializes, followed by the full U-Boot banner after a second or two.

---

### Stage 3 — OpenSBI v1.8.1 (embedded in U-Boot FIT)

RISC-V defines a Supervisor Binary Interface (SBI) — a firmware layer between
the hardware and the OS kernel that provides services (timer management,
inter-processor interrupts, system reset) without requiring the kernel to access
privileged hardware directly.

OpenSBI implements the SBI specification. It runs in M-mode (the highest RISC-V
privilege level) and drops execution to S-mode (supervisor mode) where U-Boot
proper and subsequently the OS kernel run. The kernel never runs in M-mode.

OpenSBI is not a separate flash partition — it is packaged inside the U-Boot FIT
image as a firmware blob, built in at compile time. The OpenSBI banner in the boot
log (`Platform Name: Milk-V Mars`) confirms mainline platform support with the
correct HART topology, interrupt routing, and power management for this board.

---

### Stage 4 — U-Boot proper v2026.01

With DRAM initialized and OpenSBI running, U-Boot proper takes over boot
selection. It scans the SD card for a bootable ESP, finds the FAT16 partition,
locates `EFI/BOOT/BOOTRISCV64.EFI`, and hands off via the standard UEFI boot
path — `bootefi`.

U-Boot is built with `CONFIG_EFI_VARIABLE_FILE_STORE=y` and
`CONFIG_EFI_RT_VOLATILE_STORE=y`. The first stores EFI variables in a file
(`ubootefi.var`) on the ESP rather than NVRAM, which does not exist on this SoC.
The second makes efivarfs writable at runtime, so the OS can create and modify
EFI boot entries. Without `EFI_RT_VOLATILE_STORE`, efivarfs mounts read-only
and `bootctl install` cannot register a boot entry — `bootctl` reports
"systemd-boot not installed in ESP" even when the binary is present.

The U-Boot environment persists in SPI NOR mtd1 (64K uboot-env partition).
`saveenv` at the U-Boot prompt writes there. Do not write to mtd1 directly
with `flashcp` or `dd`.

---

### Stage 5 — systemd-boot 258.7

systemd-boot is a minimal UEFI boot manager. U-Boot hands it control via
`bootefi`. It reads `loader/loader.conf` and the BLS entries in
`loader/entries/`, displays a menu (5 second timeout), and loads the selected
kernel.

On this build systemd-boot is fully installed via `bootctl install` — not just
placed at the fallback path. This means:

- The binary lives at the canonical path `EFI/systemd/systemd-bootriscv64.efi`
  in addition to the fallback `EFI/BOOT/BOOTRISCV64.EFI`
- A "Linux Boot Manager" EFI boot entry is registered in EFI Variables
  (stored in `ubootefi.var`)
- A random seed and system token are set for entropy

`bootctl status` shows the boot entry as active and in boot-order. This is the
conformant state — `dnf update kernel` will invoke `kernel-install`, which adds
new BLS entries to the ESP automatically.

---

### Stage 6 — Linux 6.19.0 (no initrd)

The kernel is built from linux-stable v6.19 using the mainline
`jh7110-milkv-mars.dts` included in mainline before v6.19. No custom DTS,
no patches — the Mars is fully supported upstream.

The rootfs is on SD card (no NVMe on this unit), so the critical built-in driver
set differs from an NVMe build. The SD card controller must be `=y` since there
is no initrd:

```
CONFIG_MMC_DW_STARFIVE=y
```

The boot entry uses an explicit device path (`root=/dev/mmcblk1p4`) rather
than UUID. Without initrd there is no early userspace to resolve UUID
references — UUID in the kernel cmdline causes an immediate panic.

Two console entries appear in the kernel cmdline:

```
console=tty0 console=ttyS0,115200
```

Linux supports multiple simultaneous console destinations. `tty0` sends kernel
messages to the HDMI framebuffer; `ttyS0` sends them to the serial UART. The
last `console=` entry is where interactive input is read, so `ttyS0` is the
interactive console. Both receive output simultaneously.

The kernel identifies the board correctly from the DTS:

```
Machine model: Milk-V Mars
```

---

### Stage 7 — Fedora 43 riscv64

Rootfs is a Fedora 43 riscv64 userspace bootstrapped using
`dnf5 --installroot --forcearch=riscv64` on a RISC-V builder VM.

Cross-arch installroot with `tsflags=noscripts` requires post-install fixups
that dnf5 cannot run in a foreign-arch chroot: ldconfig, PAM configuration,
usr-merge symlinks, crypto policy setup. These are scripted in a build script
that produces a reproducible image from scratch.

```
[root@milkv ~]# uname -a
Linux milkv 6.19.0-dirty #8 SMP Thu Mar 26 16:13:31 EDT 2026 riscv64 GNU/Linux

[root@milkv ~]# cat /etc/fedora-release
Fedora release 43 (Forty Three)

[root@milkv ~]# systemctl --failed
0 loaded units listed.

[root@milkv ~]# journalctl -b -p 3
-- No entries --
```

NetworkManager, sshd, and Cockpit (web management UI at port 9090) are running.

---

### Recovery

**SD card direct boot:** Flip the DIP switch to SD card mode (RGPIO_0=1).
The Boot ROM will load SPL from the SD card instead of SPI NOR. A valid SPL
on the SD card's first partition (with the correct type GUID) will boot the
board regardless of SPI NOR state.

**SPI NOR reflash:** With the board running, load new binaries to the FAT ESP
partition and use U-Boot's `sf` commands at the U-Boot prompt:

```
sf probe
load mmc 1:3 $kernel_addr_r u-boot-spl.bin.normal.out
sf update $kernel_addr_r 0 $filesize
load mmc 1:3 $kernel_addr_r u-boot.itb
sf update $kernel_addr_r 0x100000 $filesize
```

Never use `dd` directly to `/dev/mtdX` for SPI NOR. NOR flash requires an
erase cycle before writing — `dd` skips this and corrupts the flash silently.
`sf update` handles erase+write correctly.

---

### Open Items

- **kernel-install DTB plugin** — so `dnf update kernel` handles DTB injection
  automatically alongside the kernel image
- **tio serial profile** — `tio milkv` convenience alias (device/baud in config)
- **fw_setenv / libubootenv** — not yet in Fedora riscv64 repos; needed for
  managing U-Boot environment from userspace
