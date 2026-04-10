# Milk-V Mars — Fedora 43 riscv64 Boot Chain

A complete boot chain from SPI NOR to a running Fedora 43 riscv64 userspace
on the Milk-V Mars (StarFive JH7110). This documents what was built, the
problems encountered, and the current open items.

The goal was an open firmware stack throughout: mainline U-Boot with EFI
handoff, systemd-boot as the boot manager, current Fedora release, no vendor
binaries beyond what the JH7110 architecture requires. The result is a board
that boots to a functional Fedora userspace and can receive package updates
through normal Fedora tooling. systemd-boot is fully installed via
`bootctl install`, EFI variables are writable, and `dnf update kernel` works
without manual intervention.

**Hardware:** Milk-V Mars, StarFive JH7110 SoC (riscv64), 8GB LPDDR4,
119GB SD card (M.2 E-Key slot unpopulated on this unit).

---

## Fedora Build Toolkit

**Builder:** RISC-V builder VM — x86_64 host running riscv64 cross toolchain (Fedora 43)

**Compiler packages:**
```
gcc-riscv64-linux-gnu    # 15.2.1 (Red Hat Cross)
```

**Build dependencies:**
```
bc  bison  flex  openssl-devel  elfutils-libelf-devel  perl  python3
make  dtc  u-boot-tools
```

**Note:** `openssl-devel-engine`, `gnutls-devel`, and `python3-setuptools`
are consistently absent from a base Fedora install and must be added
explicitly.

**Rootfs bootstrap:** `dnf5 --installroot --forcearch=riscv64` on the builder VM.
Cross-arch chroot with `tsflags=noscripts` — several post-install fixups
(ldconfig, PAM, usr-merge, crypto policy) must be scripted manually since
authselect and similar tools cannot exec in a foreign-arch chroot.

---

## Source Versions

| Component    | Version | Notes / Key Config                                | Source                              |
|--------------|---------|---------------------------------------------------|-------------------------------------|
| U-Boot       | 2026.01 | EFI_LOADER=y, EFI_RT_VOLATILE_STORE=y            | github.com/u-boot/u-boot            |
| OpenSBI      | 1.8.1   | Embedded in U-Boot FIT — no separate flash stage  | Built into U-Boot                   |
| Linux        | 6.19.0  | Custom build — MMC driver built-in, no initrd     | kernel.org linux-stable             |
| systemd-boot | 258.7   | bootctl install — full BLS conformance            | Fedora 43 systemd package           |
| Fedora       | 43      | riscv64                                           | Fedora repos                        |

---

## The Boot Chain

```
JH7110 Boot ROM
  → U-Boot SPL v2026.01       (SPI NOR, mtd0, 512K)
  → OpenSBI v1.8.1            (embedded in U-Boot FIT)
  → U-Boot proper v2026.01    (SPI NOR, mtd2, 4M)
  → systemd-boot 258.7        (SD card ESP, FAT16)
  → Linux 6.19.0              (SD card ESP, custom build)
  → Fedora 43 riscv64         (SD card ext4 rootfs)
```

---

## Stage 1 — JH7110 Boot ROM

The JH7110 Boot ROM is mask ROM — burned at the factory, immutable. It is the
first code that runs after power-on. Its job is to locate a valid next-stage
image in a defined boot order and hand off.

The JH7110 Boot ROM on the Mars is controlled by DIP switches (RGPIO_0 and
RGPIO_1 exposed on the board). The factory default is SPI Flash (both switches
off), which is the production configuration.

The Boot ROM produces no UART output and introduces no observable delay. If
U-Boot SPL output does not appear on the serial console within a second or two
of power-on, the Boot ROM did not find a valid image.

**Critical:** The Boot ROM locates bootloader partitions by **type GUID**, not
by partition number. This is the sharpest edge in JH7110 building:

- SPL partition: GUID `2E54B353-1271-4842-806F-E436D6AF6985`
- U-Boot FIT partition: GUID `BC13C2FF-59E6-4262-A352-B275FD6F7172`

Using the default sgdisk type code (`8300` Linux filesystem) causes the Boot
ROM to skip the partition entirely with no error message — the board appears
dead. Silent failure, easily mistaken for hardware damage.

If SPI NOR is corrupted or empty, flip the DIP switch to SD card mode — the
Boot ROM will look for SPL on the SD card instead.

---

## Stage 2 — U-Boot SPL (SPI NOR, mtd0)

The SPL (Secondary Program Loader) is a minimal first-stage U-Boot that runs
from the SoC's on-chip SRAM — DRAM is not yet initialized. Its job is to
initialize the JH7110's DDR controller, bring up 8GB of LPDDR4, then load
U-Boot proper from SPI NOR into DRAM and jump to it.

**Build:** `starfive_visionfive2_defconfig` as base. The Mars is native in
mainline U-Boot — no patches, no custom DTS required. The SPL reads the
board's EEPROM serial string (prefix `MARS`) and matches it against a prefix
table to select the correct board DTS at runtime.

```
make CROSS_COMPILE=riscv64-linux-gnu- ARCH=riscv starfive_visionfive2_defconfig
make CROSS_COMPILE=riscv64-linux-gnu- ARCH=riscv OPENSBI=<fw_dynamic.bin>
```

Produces `u-boot-spl.bin.normal.out` (SPL) and `u-boot.itb` (FIT with OpenSBI
and U-Boot proper). Written to SPI NOR via `sf update` at the U-Boot prompt —
never via `dd` to `/dev/mtdX` (NOR flash requires an erase cycle; raw dd
corrupts silently).

---

## Stage 3 — OpenSBI v1.8.1 (embedded in U-Boot FIT)

RISC-V defines a Supervisor Binary Interface (SBI) — a firmware layer between
the hardware and the OS kernel that provides services (timer management,
inter-processor interrupts, system reset) without requiring the kernel to access
privileged hardware directly.

OpenSBI implements the SBI specification. It runs in M-mode (the highest RISC-V
privilege level) and drops the OS into S-mode (supervisor mode). The kernel
never runs in M-mode.

OpenSBI is packaged inside the U-Boot FIT image as a firmware blob, built in at
compile time — not a separate flash partition. AArch64 readers will recognize
this role as analogous to TF-A BL31 on Arm platforms. The functional position
in the boot chain is identical: secure firmware the OS trusts but never directly
controls.

The OpenSBI banner (`Platform Name: Milk-V Mars`) in the boot log confirms
mainline platform support — the Mars has a full platform definition in OpenSBI
covering the correct HART topology, interrupt routing, and power management.

---

## Stage 4 — U-Boot proper v2026.01

With DRAM initialized and OpenSBI running in M-mode, U-Boot proper takes over
boot selection. It scans the SD card ESP (p3), finds
`EFI/BOOT/BOOTRISCV64.EFI`, and hands off via standard UEFI `bootefi`.

**EFI variable storage:** `CONFIG_EFI_RT_VOLATILE_STORE=y` stores EFI
variables in `ubootefi.var` on the ESP and makes efivarfs writable at runtime.
Without this, efivarfs mounts read-only and `bootctl install` cannot register
a boot entry. `EFI_RT_VOLATILE_STORE=y` is the difference between "U-Boot has
EFI" and "U-Boot has *writable* EFI variables."

The U-Boot environment persists in SPI NOR mtd1 (64K uboot-env partition).
`saveenv` writes there. Do not write to mtd1 directly.

---

## Stage 5 — systemd-boot 258.7

systemd-boot is fully installed via `bootctl install` — not just placed at
the fallback path. This means:

- Binary at canonical path `EFI/systemd/systemd-bootriscv64.efi` and fallback
  `EFI/BOOT/BOOTRISCV64.EFI`
- "Linux Boot Manager" EFI boot entry registered in EFI Variables (`ubootefi.var`)
- `bootctl status` shows the entry active and in boot order

This is the conformant state — `dnf update kernel` invokes `kernel-install`,
which adds new BLS entries to the ESP automatically.

---

## Stage 6 — Linux 6.19.0 (no initrd)

The rootfs is on SD card rather than NVMe. The SD card controller must be
compiled in (`=y`) since there is no initrd to load it from:

```
CONFIG_MMC_DW_STARFIVE=y
```

Without initrd there is no early userspace to resolve UUID references — the
boot entry uses an explicit device path (`root=/dev/mmcblk1p4`).

**Build:**
```
make -j16 ARCH=riscv CROSS_COMPILE=riscv64-linux-gnu-
make -j16 ARCH=riscv CROSS_COMPILE=riscv64-linux-gnu- INSTALL_MOD_PATH=/tmp/modules modules_install
```

The kernel is built from linux-stable v6.19 using the mainline
`jh7110-milkv-mars.dts` — included in mainline before v6.19, no patches
required.

Two console entries in cmdline (`console=tty0 console=ttyS0,115200`) are
intentional — HDMI framebuffer and serial UART both receive kernel output
simultaneously. The last `console=` entry is the interactive console.

---

## Stage 7 — Fedora 43 riscv64

Standard Fedora 43 riscv64 userspace bootstrapped via `dnf5 --installroot
--forcearch=riscv64` on the builder VM. NetworkManager, sshd, and Cockpit
all running. The board survived a full boot cycle and `dnf upgrade` with
no intervention.

```
[root@milkv ~]# uname -a
Linux milkv 6.19.0-dirty #8 SMP Thu Mar 26 16:13:31 EDT 2026 riscv64 GNU/Linux

[root@milkv ~]# cat /etc/fedora-release
Fedora release 43 (Forty Three)

[root@milkv ~]# systemctl --failed
0 loaded units listed.
```

---

## Flash / Partition Layout

SPI NOR (via `sf update` at U-Boot prompt):

| NOR Partition | Offset    | Size | Content                    | Type GUID                              |
|---------------|-----------|------|----------------------------|----------------------------------------|
| mtd0 (spl)    | 0x000000  | 512K | U-Boot SPL                 | `2E54B353-1271-4842-806F-E436D6AF6985` |
| mtd1 (env)    | 0x080000  | 64K  | U-Boot env                 | —                                      |
| mtd2 (uboot)  | 0x100000  | 4M   | U-Boot FIT (incl. OpenSBI) | `BC13C2FF-59E6-4262-A352-B275FD6F7172` |

SD card partition table:

| Partition  | Size      | FS    | Mount | Content                              |
|------------|-----------|-------|-------|--------------------------------------|
| mmcblk1p1  | —         | —     | —     | SPL (type GUID — Boot ROM target)    |
| mmcblk1p2  | —         | —     | —     | U-Boot FIT (type GUID — Boot ROM target) |
| mmcblk1p3  | 512MB     | FAT16 | /boot | ESP — systemd-boot + kernel          |
| mmcblk1p4  | remainder | ext4  | /     | Fedora 43 rootfs                     |

---

## Recovery

**DIP switch:** Flip to SD card mode (RGPIO_0=1). Boot ROM loads SPL from
SD card p1 instead of SPI NOR, provided the partition carries the correct
type GUID.

**SPI NOR reflash** (from a running system):
```
sf probe
load mmc 1:3 $kernel_addr_r u-boot-spl.bin.normal.out
sf update $kernel_addr_r 0 $filesize
load mmc 1:3 $kernel_addr_r u-boot.itb
sf update $kernel_addr_r 0x100000 $filesize
```

Never use `dd` directly to `/dev/mtdX` — NOR flash requires an erase cycle
before writing. `sf update` handles erase+write correctly.

---

## Open Items

- **kernel-install DTB plugin** — so `dnf update kernel` handles DTB injection
  automatically alongside the kernel image
- **fw_setenv / libubootenv** — not yet in Fedora riscv64 repos; needed for
  managing U-Boot environment from userspace
- **WiFi** — AP6256 (BCM43456) on SDIO, not yet enabled in DTS

---

Questions and corrections welcome — issues for specific problems,
discussions for questions and experience sharing.
