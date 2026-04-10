# Orange Pi RV — Fedora 43 riscv64 Boot Chain

A complete boot chain from SPI NOR to a running Fedora 43 riscv64 userspace
on the Orange Pi RV (StarFive JH7110). This documents what was built, the
problems encountered, and the current open items.

The goal was an open firmware stack throughout: mainline U-Boot with EFI
handoff, systemd-boot as the boot manager, current Fedora release, no vendor
binaries beyond what the JH7110 architecture requires. The result is a board
that boots to a functional Fedora userspace and can receive package updates
through normal Fedora tooling.

**Hardware:** Orange Pi RV, StarFive JH7110 SoC (riscv64), 8GB LPDDR4,
500GB NVMe (Crucial CT500P310SSD8), 128GB SD card for ESP and recovery.

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
| Linux        | 6.19.0  | Custom build — NVMe drivers built-in, no initrd   | kernel.org linux-stable             |
| systemd-boot | 258.7   | bootctl install — full BLS conformance            | Fedora 43 systemd package           |
| Fedora       | 43      | riscv64                                           | Fedora repos                        |

---

## The Boot Chain

```
JH7110 Boot ROM
  → U-Boot SPL v2026.01       (SPI NOR, mtd0, 960K)
  → OpenSBI v1.8.1            (embedded in U-Boot FIT)
  → U-Boot proper v2026.01    (SPI NOR, mtd2, 15M)
  → systemd-boot 258.7        (NVMe ESP, FAT32)
  → Linux 6.19.0              (NVMe ESP, custom build)
  → Fedora 43 riscv64         (NVMe rootfs)
```

---

## Stage 1 — JH7110 Boot ROM

The JH7110 Boot ROM is mask ROM — burned at the factory, immutable. It runs
immediately after power-on. Its job is to locate a valid next-stage image in a
defined boot order and hand off.

For this board the Boot ROM checks SPI NOR first. If a valid U-Boot SPL is
present, it loads it into SRAM and jumps to it. If SPI NOR is empty or invalid,
it falls back to the SD card — the recovery path.

The Boot ROM produces no UART output and introduces no observable delay. If
U-Boot SPL output does not appear on the serial console within a second or two
of power-on, the Boot ROM did not find a valid image.

The JH7110 Boot ROM locates bootloader partitions by **type GUID**, not by
partition number or position. This is the sharpest edge in JH7110 building:
the SPL partition must carry GUID `2E54B353-1271-4842-806F-E436D6AF6985`
and the U-Boot FIT partition must carry `BC13C2FF-59E6-4262-A352-B275FD6F7172`.
Using the default sgdisk type code (`8300` Linux filesystem) causes the Boot
ROM to skip the partition entirely — no error message, the board appears dead.

---

## Stage 2 — U-Boot SPL (SPI NOR, mtd0)

The SPL (Secondary Program Loader) is a minimal first-stage U-Boot that runs
from the SoC's on-chip SRAM — DRAM is not yet initialized. Its job is to
initialize the JH7110's DDR controller, bring up 8GB of LPDDR4, then load
U-Boot proper from SPI NOR into DRAM and jump to it.

**Build:** `starfive_visionfive2_defconfig` as base. The Orange Pi RV shares
the JH7110 SoC with the VisionFive 2 but requires its own DTS and an EEPROM
match patch. The SPL reads the board's EEPROM serial string
(`VF7110B1-2228-...`) to select the correct DTS at runtime.

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

---

## Stage 4 — U-Boot proper v2026.01

With DRAM up and OpenSBI initialized, U-Boot proper handles boot selection.

The current `bootcmd` uses a direct `fatload`+`bootefi` from the NVMe ESP
(workaround — see Open Items). U-Boot scans NVMe, loads the kernel and DTB
into DRAM, and hands off via `bootefi`.

The U-Boot environment persists in SPI NOR mtd1 (64K uboot-env partition).
`saveenv` at the U-Boot prompt writes there. Do not write to mtd1 directly.

**EFI variable storage:** `CONFIG_EFI_RT_VOLATILE_STORE=y` makes efivarfs
writable at runtime, enabling the OS to create and modify EFI boot entries.
Without this, efivarfs mounts read-only and `bootctl install` cannot register
a boot entry.

---

## Stage 5 — Linux 6.19.0 (no initrd)

This is where JH7110 diverges most sharply from a standard Fedora kernel build.

The JH7110's PCIe controller and clock domains require specific drivers to
initialize before the NVMe can be accessed. Without an initrd, any driver in
the path between power-on and rootfs mount must be compiled in (`=y`), not as
a module (`=m`). The critical set for NVMe boot on JH7110:

```
CONFIG_PCIE_STARFIVE_HOST=y
CONFIG_CLK_STARFIVE_JH7110_STG=y
CONFIG_CLK_STARFIVE_JH7110_AON=y
CONFIG_BLK_DEV_NVME=y
CONFIG_NVME_CORE=y
CONFIG_PHY_STARFIVE_JH7110_PCIE=y
```

Incorrect config here produces a silent failure: the PCIe PHY does not
initialize, the NVMe is never enumerated, and the kernel panics at rootfs
mount with no useful diagnostics. This is not a kernel version problem, not
a DTS problem — it is a module-vs-built-in config problem. Worth stating
explicitly because the failure mode gives no indication of the actual cause.

**Build:**
```
make ARCH=riscv CROSS_COMPILE=riscv64-linux-gnu-
make ARCH=riscv CROSS_COMPILE=riscv64-linux-gnu- INSTALL_MOD_PATH=/tmp/modules modules_install
```

Custom DTS for the Orange Pi RV, based on the upstream mainline DTS that
landed after the v6.19 tag. A rebuild against v6.20 or later (when the DTS
is native) should be straightforward.

The boot entry uses an explicit device path (`root=/dev/nvme0n1p2`) rather
than UUID. Without initrd there is no early userspace to resolve UUID
references — UUID in the kernel cmdline causes an immediate panic.

---

## Stage 6 — Fedora 43 riscv64

Standard Fedora 43 riscv64 userspace bootstrapped via `dnf5 --installroot
--forcearch=riscv64` on the builder VM. NetworkManager, sshd, DNF, and SSH
all function normally. The board survived a full `dnf upgrade` and reboot
with no intervention.

```
[root@opirv ~]# uname -a
Linux opirv 6.19.0-dirty #7 SMP Fri Mar 14 ... riscv64 GNU/Linux

[root@opirv ~]# cat /etc/fedora-release
Fedora release 43 (Forty Three)
```

---

## Flash / Partition Layout

SPI NOR (via `sf update` at U-Boot prompt):

| NOR Partition | Offset    | Size  | Content                    | Type GUID                              |
|---------------|-----------|-------|----------------------------|----------------------------------------|
| mtd0 (spl)    | 0x000000  | 960K  | U-Boot SPL                 | `2E54B353-1271-4842-806F-E436D6AF6985` |
| mtd1 (env)    | 0xF00000  | 64K   | U-Boot env                 | —                                      |
| mtd2 (uboot)  | 0x1000000 | 15M   | U-Boot FIT (incl. OpenSBI) | `BC13C2FF-59E6-4262-A352-B275FD6F7172` |

NVMe partition table:

| Partition  | Size      | FS    | Mount | Content                     |
|------------|-----------|-------|-------|-----------------------------|
| nvme0n1p1  | 512MB     | FAT32 | /boot | ESP — systemd-boot + kernel |
| nvme0n1p2  | remainder | ext4  | /     | Fedora 43 rootfs            |

---

## Recovery

Insert the SD card. U-Boot's boot priority is SD card over NVMe — software
policy, no physical switch. A valid SPL and bootable ESP on the SD card will
boot the board regardless of NVMe state.

SPI NOR can be recovered via xmodem serial if the SPL is ever corrupted: hold
the BOOT button at power-on to enter Boot ROM recovery mode, then transfer a
new SPL image via the serial connection using `sx` from the `lrzsz` package.

---

## Open Items

- **systemd-boot restoration** — make it the active EFI boot target so U-Boot
  hands off via standard EFI path rather than direct `fatload`
- **kernel-install DTB plugin** — so `dnf update kernel` handles DTB injection
  automatically alongside the kernel image
- **fw_setenv / libubootenv** — not yet in the Fedora riscv64 repos; needed for
  managing U-Boot environment from userspace
- **WiFi** — AP6256 (BCM43456) on SDIO, not yet enabled in DTS

---

Questions and corrections welcome — issues for specific problems,
discussions for questions and experience sharing.
