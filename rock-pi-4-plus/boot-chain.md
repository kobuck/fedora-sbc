# Rock Pi 4 Plus — Fedora 43 aarch64 Boot Chain

A complete boot chain from eMMC raw offsets to a running Fedora 43 aarch64
userspace on the Radxa Rock Pi 4 Plus (Rockchip RK3399). This documents what
was built, the problems encountered, and the current open items.

The goal: mainline U-Boot with EFI handoff, systemd-boot as boot manager,
Fedora-packaged kernel with dracut initramfs, `dnf update kernel` working
without manual intervention. The RK3399 benefits from strong mainline support
and Fedora-packaged aarch64 kernels — this is a clean build with no custom
kernel required.

**Hardware:** Radxa Rock Pi 4 Plus v1.73, Rockchip RK3399 SoC (big.LITTLE:
2x Cortex-A72 + 4x Cortex-A53, aarch64), 4GB LPDDR4, 28.9GB onboard eMMC,
Gigabit Ethernet (RTL8211E PHY).

---

## Fedora Build Toolkit

**Builder:** AArch64 builder VM — x86_64 host running aarch64 cross toolchain (Fedora 43)

**Compiler packages:**
```
gcc-aarch64-linux-gnu    # 15.2.1 (Red Hat Cross)
gcc-arm-linux-gnu        # for TF-A M0_CROSS_COMPILE (Cortex-M0 microcontroller)
```

**Build dependencies:**
```
bc  bison  flex  openssl-devel  elfutils-libelf-devel  perl  python3
make  dtc  u-boot-tools
```

**Note:** `arm-none-eabi-gcc` (bare-metal cross compiler) is not available
in Fedora repos. `arm-linux-gnu-` works as a substitute for the TF-A
Cortex-M0 component (`M0_CROSS_COMPILE`).

**Kernel:** Fedora-packaged aarch64 kernel — `dnf install kernel` on the
target rootfs. No cross-compilation of the kernel required. `kernel-install`
and dracut initramfs generation are automatic via standard Fedora tooling.

**Rootfs bootstrap:** `dnf5 --installroot --forcearch=aarch64` on the builder VM,
or direct `dnf install kernel` in a chroot with the target ESP mounted.

---

## Source Versions

| Component    | Version                   | Notes / Key Config                                  | Source                             |
|--------------|---------------------------|-----------------------------------------------------|------------------------------------|
| TF-A         | 2.12.0                    | BL31 only — M0_CROSS_COMPILE=arm-linux-gnu-        | git.trustedfirmware.org/TF-A       |
| U-Boot       | 2026.01                   | EFI_LOADER=y, EFI_RT_VOLATILE_STORE=y              | github.com/u-boot/u-boot           |
| Linux        | 6.19.9-200.fc43.aarch64   | Fedora-packaged — dracut initramfs, UUID safe      | Fedora 43 repos                    |
| systemd-boot | 258.4                     | bootctl install — full BLS conformance             | Fedora 43 systemd package          |
| Fedora       | 43                        | aarch64                                             | Fedora repos                       |

---

## The Boot Chain

```
RK3399 Boot ROM
  → idbloader (DDR init + SPL)       (eMMC, sector 64, raw offset)
  → TF-A BL31 v2.12.0               (embedded in U-Boot FIT)
  → U-Boot proper v2026.01           (eMMC, sector 16384, raw offset)
  → systemd-boot 258.4               (eMMC ESP, FAT32)
  → Linux 6.19.9-200.fc43.aarch64   (eMMC ESP, Fedora-packaged + dracut initrd)
  → Fedora 43 aarch64               (eMMC rootfs)
```

---

## Stage 1 — RK3399 Boot ROM

The RK3399 Boot ROM is mask ROM — immutable, runs immediately after power-on.
It locates the idbloader by raw sector offset (sector 64), not by partition
type GUID. The Boot ROM reads sector 64 and, if a valid SPL signature is
present, loads and executes it from SRAM.

The Boot ROM produces no UART output. Absence of output after power-on means
the Boot ROM did not find a valid idbloader.

Fallback: if sector 64 on eMMC contains no valid image, the Boot ROM checks
the SD card slot at the same raw offset. A valid idbloader on the SD card is
sufficient to boot — no physical switch required. This is the recovery path.

---

## Stage 2 — idbloader (DDR init + SPL, eMMC sector 64)

`idbloader.img` is a compound binary: the Rockchip DDR initialization blob
prepended to the U-Boot SPL. The Boot ROM loads this into SRAM; it initializes
the RK3399's DDR controller, brings up 4GB of LPDDR4, then loads U-Boot proper
from sector 16384 into DRAM.

**Build:**
```
make CROSS_COMPILE=aarch64-linux-gnu- ARCH=arm rock-pi-4-rk3399_defconfig
# menuconfig: enable CONFIG_EFI_RT_VOLATILE_STORE=y
make CROSS_COMPILE=aarch64-linux-gnu- ARCH=arm BL31=<bl31.elf>
# Produces idbloader.img and u-boot.itb
```

**Flash** (raw offsets — always write after partitioning):
```
dd if=idbloader.img of=/dev/mmcblkX seek=64    conv=notrunc
dd if=u-boot.itb    of=/dev/mmcblkX seek=16384 conv=notrunc
```

**Important:** `sgdisk --zap-all` wipes raw offsets along with the partition
table. Always write U-Boot blobs *after* partitioning, not before.

---

## Stage 3 — TF-A BL31 v2.12.0 (embedded in U-Boot FIT)

AArch64 requires a secure world firmware running at EL3 to provide PSCI
(power state coordination) and a trusted execution environment for the OS.
TF-A BL31 fills this role: it initializes the secure monitor and drops
execution to EL2/EL1 for U-Boot proper and subsequently the OS kernel.

BL31 is embedded in the U-Boot FIT image, not a separate raw flash offset.
`arm-none-eabi-gcc` is not available in Fedora repos — use `arm-linux-gnu-`
for the Cortex-M0 microcontroller component:

```
make CROSS_COMPILE=aarch64-linux-gnu- M0_CROSS_COMPILE=arm-linux-gnu- \
     PLAT=rk3399 bl31
```

---

## Stage 4 — U-Boot proper v2026.01

With DRAM up and TF-A BL31 running in EL3, U-Boot initializes the peripheral
stack and presents a UEFI environment. It locates the ESP on eMMC (mmcblk0p1),
finds `EFI/BOOT/BOOTAA64.EFI`, and hands off via standard `bootefi`.

**Defconfig:** `rock-pi-4-rk3399_defconfig` with `CONFIG_EFI_RT_VOLATILE_STORE=y`
added. This makes efivarfs writable at runtime, enabling `bootctl install`
to register EFI boot entries.

U-Boot `boot_targets` is set to prefer eMMC (mmc0) over SD (mmc1):
```
boot_targets=mmc0 mmc1 ...
```
`bootmeths` excludes `efi_mgr` to prevent it from overriding `boot_targets`.

U-Boot identifies the board as "Rock Pi 4A" — a DTS labeling gap in mainline,
harmless.

---

## Stage 5 — systemd-boot 258.4

systemd-boot is fully installed via `bootctl install`. The "Linux Boot Manager"
EFI boot entry is registered in EFI Variables (stored in `ubootefi.var` on the
ESP via `EFI_RT_VOLATILE_STORE`). `bootctl status` confirms the entry active
and in boot order.

`dnf update kernel` invokes `kernel-install`, which adds new BLS entries to
the ESP automatically. The DTB is injected alongside the kernel via a
`kernel-install` plugin.

---

## Stage 6 — Linux 6.19.9-200.fc43.aarch64 (Fedora-packaged)

The kernel is the standard Fedora-packaged aarch64 kernel — installed via
`dnf install kernel` on the target rootfs. No custom build required; RK3399
and the Rock Pi 4B Plus DTS are fully supported in mainline and in the Fedora
kernel package.

With a dracut initramfs, modules do not need to be compiled in. UUID syntax
in BLS boot entries is safe — dracut resolves it in the initrd before the
real rootfs is mounted.

Benign boot messages (not errors):
- DMC OPP table missing — affects DRAM frequency scaling, not function
- PCIe timeout — no PCIe device present on this build, timeout is expected
- VLAN restore — cosmetic, network is unaffected

**kernel-install notes:** The Fedora vmlinuz path is
`/usr/lib/modules/<ver>/vmlinuz`, not `/boot/`. The target directory on the
FAT ESP must be pre-created before `kernel-install` runs in a chroot context —
it will not create it. Mount the ESP directly (not via udisks) for writes
inside a chroot.

```
[root@rockpi4 ~]# uname -a
Linux rockpi4 6.19.9-200.fc43.aarch64 #1 SMP ... aarch64 GNU/Linux

[root@rockpi4 ~]# cat /etc/fedora-release
Fedora release 43 (Forty Three)

[root@rockpi4 ~]# systemctl --failed
0 loaded units listed.
```

---

## Stage 7 — Fedora 43 aarch64

Standard Fedora 43 aarch64 userspace. Rootfs on onboard eMMC (mmcblk0p2).
NetworkManager, sshd, chrony, and Cockpit all running. Full `dnf upgrade`
survived with no intervention.

A fresh Fedora rootfs has `PermitRootLogin` disabled by default in sshd —
enable it explicitly in `sshd_config` if root SSH access is required.

---

## Flash / Partition Layout

eMMC raw offsets (written with `dd` — these precede the partition table):

| Offset (sectors) | Content        | Notes                        |
|------------------|----------------|------------------------------|
| 64               | idbloader.img  | DDR init + SPL               |
| 16384            | u-boot.itb     | U-Boot proper + TF-A BL31   |

eMMC partition table:

| Partition  | Size     | FS    | Mount | Content                             |
|------------|----------|-------|-------|-------------------------------------|
| mmcblk0p1  | 1GB      | FAT32 | /boot | ESP — systemd-boot + kernels + DTBs |
| mmcblk0p2  | ~27.9GB  | ext4  | /     | Fedora 43 rootfs                    |

SD card retained as fallback — Boot ROM falls back to SD sector 64 if eMMC
idbloader is absent or invalid.

---

## Recovery

The Boot ROM checks the SD card slot at sector 64 if eMMC contains no valid
image. A valid idbloader on the SD card is sufficient to boot — no physical
switch required.

From U-Boot prompt (if SD card is available as mmc1):
```
load mmc 1:1 $kernel_addr_r idbloader.img
mmc dev 0; mmc write $kernel_addr_r 0x40 <blkcount>
load mmc 1:1 $kernel_addr_r u-boot.itb
mmc write $kernel_addr_r 0x4000 <blkcount>
```

---

## Open Items

- **WiFi** — AP6256 (BCM4345C5); firmware not yet installed

---

Questions and corrections welcome — issues for specific problems,
discussions for questions and experience sharing.
