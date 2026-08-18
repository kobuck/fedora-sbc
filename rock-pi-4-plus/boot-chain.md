# Rock Pi 4 Plus — Fedora 44 aarch64 Boot Chain

A complete boot chain from SD card raw offsets to a running Fedora 44 aarch64
userspace on the Radxa Rock Pi 4 Plus (Rockchip RK3399). This documents what
was built, the problems encountered, and the current open items.

The goal: mainline U-Boot with EFI handoff, systemd-boot as boot manager,
Fedora-packaged kernel with dracut initramfs, `dnf update kernel` working
without manual intervention. **This goal is now fully validated** — an
in-place `dnf update kernel` (7.1.4 → 7.1.8) completed with zero manual
steps: correct BLS entry, devicetree line, and root UUID were all generated
automatically, and `loader.conf: default *` promoted the new kernel without
any operator action. The RK3399 benefits from strong mainline support and
Fedora-packaged aarch64 kernels — this is a clean build with no custom
kernel required.

**Hardware:** Radxa Rock Pi 4 Plus v1.73, Rockchip RK3399 SoC (big.LITTLE:
2x Cortex-A72 + 4x Cortex-A53, aarch64), 4GB LPDDR4, 28.9GB onboard eMMC,
Gigabit Ethernet (RTL8211E PHY).

**Two build targets from one image:** this board has both an SD slot and
onboard eMMC. The build is validated on SD first, then the *same* validated
image is cloned onto eMMC (only storage-level identifiers — GPT GUIDs,
filesystem UUIDs — are re-randomized; machine-id, hostname, and SSH host
keys are deliberately shared, since only one device is ever the actual
running system at a time). See [GPT Image Creation](../gpt-image-creation.md)
for the general process this follows, and the Two-Target Build section below
for specifics.

---

## Fedora Build Toolkit

**Builder:** AArch64 builder VM — x86_64 host running aarch64 cross toolchain (Fedora 44)

**Compiler packages:**
```
gcc-aarch64-linux-gnu    # cross toolchain
gcc-arm-linux-gnu        # for TF-A M0_CROSS_COMPILE (Cortex-M0 microcontroller)
```

**Build dependencies:**
```
bc  bison  flex  openssl-devel  elfutils-libelf-devel  perl  python3
make  dtc  u-boot-tools  gdisk
```

**Note:** `arm-none-eabi-gcc` (bare-metal cross compiler) is not available
in Fedora repos. `arm-linux-gnu-` works as a substitute for the TF-A
Cortex-M0 component (`M0_CROSS_COMPILE`).

**Kernel:** Fedora-packaged aarch64 kernel — `dnf install kernel` on the
target rootfs. No cross-compilation of the kernel required. `kernel-install`
and dracut initramfs generation are automatic via standard Fedora tooling,
**with two required pre-existing config files** — see Stage 6 below.

**Rootfs bootstrap:** `dnf5 --installroot --forcearch=aarch64` on the builder VM
under qemu-binfmt emulation, assembled into a real partitioned image (not a
flat chroot) before `kernel-install` is run — see
[GPT Image Creation](../gpt-image-creation.md) for why this ordering matters.

---

## Source Versions

| Component    | Version                   | Notes / Key Config                                  | Source                             |
|--------------|---------------------------|-------------------------------------------------------|------------------------------------|
| TF-A         | 2.15.0                    | BL31 only — M0_CROSS_COMPILE=arm-linux-gnu-        | git.trustedfirmware.org/TF-A       |
| U-Boot       | 2026.07                   | EFI_LOADER=y and EFI_VARIABLE_FILE_STORE=y now default-on; EFI_RT_VOLATILE_STORE=y still manual | github.com/u-boot/u-boot |
| Linux        | 7.1.4-200.fc44.aarch64 → 7.1.8 confirmed via in-place `dnf update` | Fedora-packaged — dracut initramfs, UUID safe | Fedora 44 repos |
| systemd-boot | 259.7 → 259.8 (via same `dnf update`) | bootctl install — full BLS conformance | Fedora 44 systemd package |
| Fedora       | 44                        | aarch64                                             | Fedora repos                       |

---

## The Boot Chain

```
RK3399 Boot ROM
  → idbloader (DDR init + SPL)       (SD card, sector 64, raw offset)
  → TF-A BL31 v2.15.0               (embedded in U-Boot FIT)
  → U-Boot proper v2026.07           (SD card, sector 16384, raw offset)
  → systemd-boot 259.7               (SD card ESP, FAT32)
  → Linux 7.1.4-200.fc44.aarch64    (SD card ESP, Fedora-packaged + dracut initrd)
  → Fedora 44 aarch64               (SD card rootfs; cloned to eMMC as a
                                       second, independently-bootable target)
```

---

## Stage 1 — RK3399 Boot ROM

The RK3399 Boot ROM is mask ROM — immutable, runs immediately after power-on.
It locates the idbloader by raw sector offset (sector 64), not by partition
type GUID. The Boot ROM reads sector 64 and, if a valid SPL signature is
present, loads and executes it from SRAM.

The Boot ROM produces no UART output. Absence of output after power-on means
the Boot ROM did not find a valid idbloader.

The Boot ROM's own device-selection behavior is fixed — it always checks the
same physical slot(s) in the same order. **Device selection *after* the Boot
ROM stage (i.e. which of SD or eMMC U-Boot ultimately boots from) is governed
by U-Boot's own environment, not the Boot ROM** — see Stage 4.

---

## Stage 2 — idbloader (DDR init + SPL, SD card sector 64)

`idbloader.img` is a compound binary: the Rockchip DDR initialization blob
prepended to the U-Boot SPL. The Boot ROM loads this into SRAM; it initializes
the RK3399's DDR controller, brings up 4GB of LPDDR4, then loads U-Boot proper
from sector 16384 into DRAM.

**Build:**
```
make CROSS_COMPILE=aarch64-linux-gnu- ARCH=arm rock-pi-4-rk3399_defconfig
# menuconfig: enable CONFIG_EFI_RT_VOLATILE_STORE=y (only manual addition needed
# on v2026.07 — EFI_LOADER and EFI_VARIABLE_FILE_STORE are on by default now)
make CROSS_COMPILE=aarch64-linux-gnu- ARCH=arm BL31=<bl31.elf>
# Produces idbloader.img and u-boot.itb
```

**Flash** (raw offsets — always written *after* partitioning and rootfs
assembly are fully complete, never before):
```
dd if=idbloader.img of=/dev/sdX seek=64    conv=notrunc bs=512
dd if=u-boot.itb    of=/dev/sdX seek=16384 conv=notrunc bs=512
```

**Important:** `sgdisk --zap-all` wipes raw offsets along with the partition
table. Always write U-Boot blobs *after* partitioning. Also: a bare
`sgdisk -n 1:64:16383` will silently move the partition start to sector 2048
(sgdisk's default 2048-sector alignment) — this breaks the boot chain since
sector 64 must remain unpartitioned/reserved. Use `sgdisk -a 1 -n 1:64:16383`
to force exact-sector placement. See
[GPT Image Creation](../gpt-image-creation.md) for the full partition layout.

---

## Stage 3 — TF-A BL31 v2.15.0 (embedded in U-Boot FIT)

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

## Stage 4 — U-Boot proper v2026.07

With DRAM up and TF-A BL31 running in EL3, U-Boot initializes the peripheral
stack and presents a UEFI environment. It locates the ESP, finds
`EFI/BOOT/BOOTAA64.EFI`, and hands off via standard `bootefi`.

**Defconfig:** `rock-pi-4-rk3399_defconfig` with `CONFIG_EFI_RT_VOLATILE_STORE=y`
added (the other two EFI options are on by default in v2026.07). This makes
efivarfs writable at runtime, enabling `bootctl install` to register EFI
boot entries.

**Device selection — a real gotcha once a board has two bootable devices.**
`bootcmd=bootflow scan -lb` uses U-Boot's bootmeth *priority* system, not a
strict walk of `boot_targets` in order. With `efi_mgr` present in the
compiled-in default `bootmeths` list, U-Boot scans for *any* device with a
valid bootable ESP and boots the first one it finds — this can mean it
boots eMMC even when the SD card is physically present and
`boot_targets=mmc1 mmc0 ...` (SD first) is already set correctly, because
`efi_mgr`'s EFI-Boot-Manager-style discovery doesn't respect that ordering.

**Fix — set once both devices are bootable, persisted via `saveenv`:**
```
setenv bootmeths "extlinux script efi pxe"
saveenv
```
Removing `efi_mgr` forces U-Boot to walk `boot_targets` in the order given,
using the remaining bootmeths. This is a one-time environment change, not a
per-boot workaround — it survives reboots via the saved environment.

U-Boot identifies the board as "Rock Pi 4A" (`fdtfile=rockchip/
rk3399-rock-pi-4a.dtb` in the default env) — a DTS labeling gap in
mainline's U-Boot defconfig, harmless; it does not affect which DTB the
*kernel* actually uses (see Stage 6).

---

## Stage 5 — systemd-boot 259.7

systemd-boot is installed by placing `systemd-bootaa64.efi` (from the
target-arch `systemd-boot-unsigned` RPM — **not** `bootctl install` run from
the x86_64 builder VM, which would install the wrong-architecture binary)
at both `EFI/systemd/systemd-bootaa64.efi` and `EFI/BOOT/BOOTAA64.EFI`.
`loader.conf` is written with `default *` (not a hardcoded kernel filename)
so any newly-installed kernel automatically becomes the boot default with
no manual `bootctl set-default` step ever required.

`dnf update kernel` invokes `kernel-install`, which adds new BLS entries to
the ESP automatically, including the devicetree line — see Stage 6 for the
two config files that make this work correctly.

---

## Stage 6 — Linux 7.1.4-200.fc44.aarch64 (Fedora-packaged) — `dnf update kernel` validated

The kernel is the standard Fedora-packaged aarch64 kernel — installed via
`dnf install kernel` on the target rootfs. No custom build required; RK3399
and the Rock Pi 4B Plus DTS are fully supported in mainline and in the Fedora
kernel package.

**Two config files are required for `kernel-install` to produce a fully
correct BLS entry, and both must exist *before* the first `kernel-install`
run:**

1. **`/etc/kernel/devicetree`** — a plain text file containing the DTB's
   relative path, e.g. `rockchip/rk3399-rock-pi-4b-plus.dtb`.
   `kernel-install` does **not** auto-detect the board's DTB; without this
   file, the BLS entry is written with no `devicetree` line at all, and the
   kernel will not boot correctly. This is not a `kernel-install` defect —
   it's a required, board-specific config file that must be present in the
   built rootfs.
2. **`/etc/kernel/cmdline`** — required *specifically* when running
   `kernel-install` inside a cross-arch qemu-binfmt chroot on a builder VM
   (not needed for `kernel-install` runs on the live, already-booted target,
   e.g. a later `dnf update kernel`). Without it, `kernel-install` falls
   back to reading the **host's** live `/proc/cmdline` — since a bare
   `chroot` shares the host kernel/PID namespace, this silently produces a
   BLS entry with the *builder VM's own* root UUID and machine-id baked in,
   which looks entirely plausible (valid UUID/machine-id format) but boots
   the wrong identity. Write this file with the target's correct
   `root=UUID=... console=... systemd.machine_id=...` before the first
   in-chroot `kernel-install` run.

With both files in place, a subsequent `dnf update kernel` on the live,
booted system — no chroot involved — produces a fully correct BLS entry
automatically, as confirmed:

```
[root@rockpi4 ~]# dnf upgrade --refresh
...
kernel-core.aarch64      7.1.8-200.fc44   updates
...
[root@rockpi4 ~]# cat /boot/efi/loader/entries/<machine-id>-7.1.8-200.fc44.aarch64.conf
title      Fedora Linux 44 (Forty Four)
version    7.1.8-200.fc44.aarch64
machine-id <machine-id>
options    root=UUID=<correct-root-uuid> rw rootwait console=ttyS2,1500000 earlycon systemd.machine_id=<machine-id>
linux      /<machine-id>/7.1.8-200.fc44.aarch64/linux
devicetree /<machine-id>/7.1.8-200.fc44.aarch64/rk3399-rock-pi-4b-plus.dtb
initrd     /<machine-id>/7.1.8-200.fc44.aarch64/initrd
```

Correct devicetree line, correct root UUID, correct machine-id — all
generated with zero manual intervention, and `bootctl` immediately shows the
new kernel as the default (due to `loader.conf: default *`).

With a dracut initramfs, modules do not need to be compiled in. UUID syntax
in BLS boot entries is safe — dracut resolves it in the initrd before the
real rootfs is mounted.

Benign boot messages (not errors):
- `rk3288-crypto`/`rk-sha1`/`rk-sha256`/`rk-md5` self-test failures
  (`export() overran state buffer`) — the hardware crypto accelerator's
  kernel self-test fails and the kernel falls back to software crypto;
  common on this SoC/kernel combination, does not affect boot or function
- DMC OPP table missing — affects DRAM frequency scaling, not function
- PCIe timeout — no PCIe device present on this build, timeout is expected
- VLAN restore — cosmetic, network is unaffected

**kernel-install notes:** The Fedora vmlinuz path is
`/usr/lib/modules/<ver>/vmlinuz`, not `/boot/`. `/boot/efi/loader/entries/`
must be pre-created before the first `kernel-install` run in a chroot
context — it will not create it. Mount the ESP directly (not via udisks)
for writes inside a chroot.

```
[root@rockpi4 ~]# uname -a
Linux rockpi4 7.1.4-200.fc44.aarch64 #1 SMP ... aarch64 GNU/Linux

[root@rockpi4 ~]# cat /etc/fedora-release
Fedora release 44 (Forty Four)

[root@rockpi4 ~]# systemctl --failed
  UNIT              LOAD   ACTIVE SUB    DESCRIPTION
● logrotate.service loaded failed failed Rotate log files
2 loaded units listed.
```
(The `logrotate.service` failure is benign — the `psacct` package is not
installed, so `/var/account/pacct` doesn't exist for that one logrotate
stanza to rotate. Does not affect anything functional.)

---

## Stage 7 — Fedora 44 aarch64

Standard Fedora 44 aarch64 userspace. SD card is the primary/validated
build; the identical image is also cloned onto onboard eMMC as a second,
independently-bootable target (see Two-Target Build below).
NetworkManager, sshd, chrony, and Cockpit all running. Full `dnf upgrade`
survived with no intervention on the SD build.

A fresh Fedora rootfs has root password locked (`!locked`) and SSH key-only
access configured (`PermitRootLogin prohibit-password`,
`PasswordAuthentication no`) — this is a deliberate access policy, not an
oversight; console/password login is intentionally unavailable.

Two config gaps were found and fixed on first live boot, worth checking on
any freshly-built rootfs before relying on it:
- **CA trust bundle never generated** — `/etc/pki/ca-trust/extracted/pem/
  tls-ca-bundle.pem` did not exist, blocking all HTTPS-based `dnf`
  operations. Fix: run `update-ca-trust extract` once, live.
- **firewalld's PolicyKit action symlink missing** —
  `/usr/share/polkit-1/actions/org.fedoraproject.FirewallD1.policy` (the
  file polkit actually resolves for firewalld D-Bus actions) was absent
  from disk despite the RPM database listing it as owned by the package —
  a scriptlet/symlink-selection step that apparently doesn't complete
  correctly when the package installs under qemu-binfmt emulation. Fix:
  `dnf reinstall firewalld` once running live (not in chroot).

---

## Two-Target Build — SD (primary) + eMMC (cloned)

This board has both an SD card slot and 28.9GB onboard eMMC. Rather than
building each independently (which previously introduced drift and
non-reproducible states between the two), the build is validated once on
SD, then the identical, already-proven image is cloned onto eMMC:

1. RootFS partition size is **fixed** (26 GiB), not "remainder" — sized to
   fit within eMMC's actual usable capacity (measured via
   `/sys/class/block/<dev>/size`, not the nominal/marketing capacity) with
   margin, so the same partition table works unmodified on both targets.
2. Clone is executed **live**, from the already-booted SD system, treating
   the (soldered, non-removable) eMMC as an inert target block device
   visible alongside the running SD boot. There is no external eMMC
   programmer for this board — a live, already-bootable OS is the only way
   to reach it at all.
3. **Storage-level identifiers are re-randomized on the eMMC copy** (GPT
   disk GUID, all partition GUIDs, ext4 filesystem UUID, vfat volume
   serial) — required because both devices are simultaneously visible as
   block devices to whichever OS is running, regardless of which one
   actually booted, so UUID-based resolution (`blkid`, `/etc/fstab`, BLS
   `options` lines) must disambiguate correctly.
4. **Machine-level identity is deliberately shared** (machine-id, SSH host
   keys, hostname) — since only one of the two devices is ever the actual
   running system at any moment, there is no scenario requiring two
   distinct machine identities to coexist.
5. `/etc/fstab`, every BLS entry's `options root=UUID=...` line, and
   `/etc/kernel/cmdline` must all be updated on the eMMC copy to reference
   its new (randomized) root UUID — the third file (`/etc/kernel/cmdline`)
   is easy to miss but matters for any *future* `kernel-install` run
   against the eMMC copy to pick up the correct identity rather than a
   stale one.

See [GPT Image Creation](../gpt-image-creation.md) for the general clone
procedure this follows.

---

## Flash / Partition Layout

Raw offsets (written with `dd`, always as the final build step — these
sectors sit *within* the fenced partition ranges below:

| Offset (sectors) | Content        | Notes                        |
|------------------|----------------|-------------------------------|
| 64               | idbloader.img  | DDR init + SPL               |
| 16384            | u-boot.itb     | U-Boot proper + TF-A BL31    |

Partition table (5 partitions — SPL+/U-Boot/RESERVED are protective fence
entries only, containing no filesystem; hardware ignores their GPT type
GUIDs on this SoC, they exist purely so disk tools are aware of these
raw regions):

| Partition | Sector range        | Size    | FS    | Mount     | Content                 |
|-----------|----------------------|---------|-------|-----------|--------------------------|
| p1 SPL+   | 64 – 16383           | ~8 MB   | —     | —         | fence over idbloader zone |
| p2 UBOOT  | 16384 – 24575        | 4 MB    | —     | —         | fence over u-boot.itb zone |
| p3 RESERVED | 24576 – 32767      | 4 MB    | —     | —         | unoccupied fence |
| p4 ESP    | 32768 – 4227071      | 2 GB    | FAT32 | /boot/efi | systemd-boot + kernels + DTBs |
| p5 RootFS | 4227072 – (26 GiB fixed) | 26 GiB | ext4 | / | Fedora 44 rootfs |

Full detail, GUIDs, and the general build procedure this follows:
[GPT Image Creation](../gpt-image-creation.md).

---

## Recovery

The Boot ROM checks the SD card slot at sector 64 if eMMC contains no valid
image (or vice versa, per U-Boot's own `boot_targets` order once past the
Boot ROM stage). With SD as the primary validated target, pulling the SD
card and power-cycling falls through to eMMC (once `bootmeths` is set as
described in Stage 4) — this is the standard rollback path used during
development: build/validate on SD, test eMMC by removing the SD, reinsert
SD immediately if eMMC fails.

---

## Open Items

- **WiFi** — AP6256 (BCM4345C5); firmware not yet installed
- **eMMC SELinux regression after `dnf upgrade`** — an in-place kernel
  upgrade (7.1.4 → 7.1.8) that completed cleanly on the SD build instead
  left the eMMC clone unable to complete boot on *either* kernel: the new
  kernel (7.1.8) hangs after the display driver stack initializes; the
  previously-working kernel (7.1.4) now also fails, with SELinux (enforcing
  mode) denying `systemd` PID 1 a required early write, causing a hard
  freeze. The SD build was never subjected to the same upgrade and remains
  fully healthy. Root cause not yet identified — likely related to a
  `selinux-policy`/`systemd` version pairing bumped together in the same
  transaction, but this has not been confirmed. Filed as an open
  investigation, not yet a bug report against any specific upstream
  project, pending root-cause work.

---

Questions and corrections welcome — issues for specific problems,
discussions for questions and experience sharing.
