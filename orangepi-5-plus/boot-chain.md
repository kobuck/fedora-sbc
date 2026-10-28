# Orange Pi 5 Plus — Fedora 44 aarch64 Boot Chain

A complete boot chain from SD card raw offsets to a running Fedora 44
aarch64 userspace on the Orange Pi 5 Plus (Rockchip RK3588), with the root
filesystem on an NVMe SSD. This documents what was built and how.

The goal: mainline U-Boot with EFI handoff, systemd-boot as boot manager,
Fedora-packaged kernel with dracut initramfs — same proven pattern already
validated on this lab's RK3399 (Rock Pi 4) and RK3566 (ASUS Tinker Board 3)
builds, extended here to RK3588. The RK3588 has solid mainline support;
this is a clean build with no custom kernel patches required.

**Hardware:** Orange Pi 5 Plus, Rockchip RK3588 SoC (big.LITTLE: 4x
Cortex-A76 + 4x Cortex-A55, aarch64), 4GB LPDDR4/4X, dual Realtek RTL8125
2.5GbE NICs (PCIe), onboard eMMC socket, user-added NVMe SSD (PCIe), 16MB
SPI-NOR (holds the vendor bootloader, untouched by this build), 3-pin
dedicated TTL serial header.

**Boot device is SD; root filesystem is NVMe.** Unlike this lab's other
split-storage builds (e.g. Rock Pi 4's SD-or-eMMC pattern, where the same
image is cloned onto a second *boot-capable* device), the Orange Pi 5
Plus's RK3588 Boot ROM **cannot start SPL from NVMe under any
circumstance** — PCIe/NVMe drivers exist only in U-Boot proper, never in
the Boot ROM's TPL/SPL stage. This is an architectural property of the
SoC, not a configuration limitation. The build therefore boots from SD
(firmware + kernel + initrd on the SD's ESP) and mounts root from the NVMe
SSD once U-Boot/the kernel are far enough along to have PCIe/NVMe support.
The onboard eMMC socket is present but not used as a boot target on this
specific board (see Hardware Note below).

---

## Fedora Build Toolkit

**Builder:** AArch64 builder VM — x86_64 host running aarch64 cross
toolchain (Fedora 44)

**Compiler packages:**
```
gcc-aarch64-linux-gnu    # cross toolchain
gcc-arm-linux-gnu        # for TF-A M0_CROSS_COMPILE (Cortex-M0 microcontroller)
```

**Build dependencies:**
```
bc  bison  flex  openssl-devel  elfutils-libelf-devel  perl  python3
make  dtc  u-boot-tools  gdisk  ncurses-devel  dwarves
```

**Kernel:** Fedora-packaged aarch64 kernel. No cross-compilation of the
kernel source required — built in-tree against the same source used to
produce the Fedora RPM, with `CONFIG_DEBUG_INFO_NONE=y` recommended over
the distro default full-DWARF debug info (saves significant build-host
disk space with no practical downside for a board-bringup kernel).

**Rootfs bootstrap:** `dnf5 --installroot --forcearch=aarch64` on the
builder VM under qemu-binfmt emulation, assembled into a real partitioned
image before `kernel-install` is run.

**DDR init blob:** Rockchip's RK3588 requires a proprietary DDR
initialization binary (from Rockchip's `rkbin` releases) as a build
*input* to U-Boot's own SPL build — combined with the compiled SPL into
`idbloader.img`. This board uses the LPDDR4/4X variant
(`rk3588_ddr_lp4_2112MHz_lp5_2400MHz_v1.24.bin`), confirmed against the
board's actual schematic (two-channel LPDDR4/4X package, not the later
LPDDR5 hardware revision).

---

## Source Versions

| Component    | Version   | Notes / Key Config                                  | Source |
|--------------|-----------|------------------------------------------------------|--------|
| TF-A         | 2.15.0    | BL31 only — M0_CROSS_COMPILE=arm-linux-gnu-          | git.trustedfirmware.org/TF-A |
| rkbin (DDR blob) | v1.24 (rk3588_ddr_lp4_2112MHz_lp5_2400MHz) | Build input to U-Boot's SPL, not a lab source build | github.com/rockchip-linux/rkbin |
| U-Boot       | 2026.07   | EFI_LOADER, EFI_VARIABLE_FILE_STORE default-on; EFI_RT_VOLATILE_STORE set manually | github.com/u-boot/u-boot |
| Linux        | 7.2.x-fc44.aarch64 | Fedora-packaged — dracut initramfs, UUID safe | Fedora 44 repos |
| systemd-boot | 259.x     | bootctl install — full BLS conformance | Fedora 44 systemd package |
| Fedora       | 44        | aarch64 | Fedora repos |

---

## The Boot Chain

```
RK3588 Boot ROM
  → idbloader (DDR init blob + SPL)   (SD card, sector 64, raw offset)
  → TF-A BL31 v2.15.0                 (embedded in U-Boot FIT)
  → U-Boot proper v2026.07            (SD card, sector 16384, raw offset)
  → systemd-boot                      (SD card ESP, FAT32)
  → Linux 7.2.x-fc44.aarch64          (SD card ESP, Fedora-packaged + dracut initrd)
  → Fedora 44 aarch64 rootfs          (NVMe SSD — root mounted after kernel init)
```

---

## Stage 1 — RK3588 Boot ROM

Mask ROM, immutable, runs immediately after power-on. Locates the
idbloader by raw sector offset (sector 64) on whichever device the Boot
ROM's fixed scan order checks first — same mechanism as this lab's RK3399
(Rock Pi 4) and RK3566 (ASUS Tinker Board 3) builds. The Boot ROM produces no
UART output; absence of output after power-on means no valid idbloader was
found at that offset.

**Boot device priority is empirically SD > onboard SPI-NOR** on this
board (confirmed via LED-color boot-source test): the Boot ROM finds the
SD card's idbloader first when an SD card is present, and only falls
through to the onboard SPI-NOR (which ships from the factory holding the
vendor's own U-Boot, not blank) when no SD card is inserted.

---

## Stage 2 — idbloader (DDR init + SPL, SD card sector 64)

A compound binary: the Rockchip DDR initialization blob prepended to the
U-Boot SPL. Initializes the RK3588's DDR controller (LPDDR4/4X trained at
2112MHz on this board) and loads U-Boot proper from sector 16384.

**Build:**
```
make CROSS_COMPILE=aarch64-linux-gnu- orangepi-5-plus-rk3588_defconfig
# Verify explicitly (don't assume defconfig defaults):
#   CONFIG_EFI_LOADER=y
#   CONFIG_EFI_VARIABLE_FILE_STORE=y
#   CONFIG_EFI_RT_VOLATILE_STORE=y   (add manually if not already set)
make CROSS_COMPILE=aarch64-linux-gnu- BL31=<bl31.elf> \
     ROCKCHIP_TPL=<rk3588_ddr_lp4_2112MHz_lp5_2400MHz_v1.24.bin>
# Produces idbloader.img and u-boot.itb
```

**The `ROCKCHIP_TPL=` argument is required** — omitting it produces a
silently 0-byte `idbloader.img` with the build's own error output
explaining the missing external DDR blob. This is RK3588-specific; the
RK3399 build this lab's process is otherwise modeled on does not need an
external TPL/DDR blob in the same way.

**Flash** (raw offsets, written only after partitioning and rootfs
assembly are complete):
```
dd if=idbloader.img of=/dev/sdX seek=64    conv=notrunc bs=512
dd if=u-boot.itb    of=/dev/sdX seek=16384 conv=notrunc bs=512
```

---

## Stage 3 — TF-A BL31 v2.15.0 (embedded in U-Boot FIT)

Provides PSCI and the secure-world execution environment required at
EL3, dropping execution to EL2/EL1 for U-Boot and the OS kernel. Embedded
in the U-Boot FIT image, not a separate raw flash offset.

```
make CROSS_COMPILE=aarch64-linux-gnu- M0_CROSS_COMPILE=arm-linux-gnu- \
     PLAT=rk3588 DEBUG=0 bl31
```

---

## Stage 4 — U-Boot proper v2026.07

With DRAM up and TF-A BL31 running in EL3, U-Boot initializes the
peripheral stack (including PCIe, required to see the NVMe SSD) and
presents a UEFI environment, locating the ESP and handing off via
`bootefi`.

**Defconfig:** `orangepi-5-plus-rk3588_defconfig` with
`CONFIG_EFI_RT_VOLATILE_STORE=y` added explicitly.

**NVMe appears in `boot_targets`** alongside `mmc1 mmc0` once U-Boot
proper is running (PCIe/NVMe support exists at this stage, unlike the
Boot ROM/SPL stage) — this is what allows root-on-NVMe to work even
though boot-from-NVMe does not.

**Note:** this board's U-Boot build has no persistent environment storage
backend configured (no `CONFIG_ENV_IS_IN_*` option set) — the U-Boot
environment lives in RAM only and does not survive a reset. `saveenv`
is not expected to persist anything on this build; any environment
variable needing to be checked must be read live at the `=>` console
prompt.

---

## Stage 5 — systemd-boot

Installed by placing the target-arch `systemd-bootaa64.efi` (from the
`systemd-boot-unsigned` RPM, not built by `bootctl install` on the
x86_64 builder) at both `EFI/systemd/systemd-bootaa64.efi` and
`EFI/BOOT/BOOTAA64.EFI`. `loader.conf` uses `default *` so any newly
installed kernel becomes the boot default automatically.

The ESP is mounted at `/efi` (this lab's standard convention, matching
`bootctl install`'s own native default).

---

## Stage 6 — Linux (Fedora-packaged), root on NVMe

Standard Fedora-packaged aarch64 kernel. As with this lab's other
Rockchip builds, two config files must exist before the first
`kernel-install` run:

1. **`/etc/kernel/devicetree`** — relative DTB path
   (`rockchip/rk3588-orangepi-5-plus.dtb`); `kernel-install` does not
   auto-detect the board's DTB.
2. **`/etc/kernel/cmdline`** — required specifically for
   `kernel-install` runs inside a cross-arch qemu-binfmt chroot, so the
   generated BLS entry carries the *target's* correct root UUID and
   machine-id rather than silently inheriting the builder VM's own live
   `/proc/cmdline`.

The kernel's `root=UUID=...` in the BLS entry points at the **NVMe's**
root partition UUID, not the SD card — the SD only carries the ESP
(firmware, kernel, initrd, DTB); the kernel mounts root from NVMe once
dracut's initrd has resolved that UUID against the now-PCIe-enumerated
NVMe device.

**IPv6 addressing note:** a from-scratch rootfs bootstrap does not ship a
persisted NetworkManager connection profile, so NetworkManager's
auto-generated in-memory profile defaults to `stable-privacy`
(randomized) IPv6 address generation rather than the kernel's own
EUI-64 default — the two settings are independent, and `nmcli`'s status
output does not distinguish them from the outside. If stable, MAC-derived
IPv6 addressing is required (e.g. for DNS infrastructure that computes
addresses from a known MAC), set it explicitly and let it write a
persisted keyfile:
```
nmcli connection modify "<connection-name>" ipv6.addr-gen-mode eui64
```

---

## Stage 7 — Fedora 44 aarch64 (root on NVMe)

Standard Fedora 44 aarch64 userspace: NetworkManager, sshd, chronyd,
firewalld, Cockpit. Root password locked, SSH key-only access
(`PermitRootLogin prohibit-password`, `PasswordAuthentication no`).

A `dnf5 --installroot` bootstrap with `tsflags=noscripts` (required for
cross-arch assembly) skips every package's install/postinstall
scriptlets across the transaction — not just the obviously-affected ones.
Confirmed required remediation on first live boot:

- **`authselect select local`** — never run automatically; without it,
  `pam_systemd.so` is absent from the PAM session stack, and
  `user@<uid>.service` fails with `$XDG_RUNTIME_DIR is not set`.
- **CA trust bundle** — `update-ca-trust extract` must be run once;
  without it, all HTTPS access (including `dnf`) fails with a missing
  trust-anchor file error.
- **firewalld PolicyKit symlink** — `dnf reinstall firewalld` restores
  a symlink-selection step that doesn't complete correctly under
  qemu-binfmt emulation; without it, every `firewall-cmd` call fails.
- **`/etc/selinux/config`** — not written by the noscripts bootstrap at
  all. If Enforcing mode is wanted (this lab's standard, matching stock
  Fedora default), set it explicitly and force a full relabel — see the
  warning below, this step has a real failure mode if done carelessly.
- **cockpit SELinux policy module** — shipped inside
  `selinux-policy-targeted` but not loaded into the active policy by a
  noscripts bootstrap; `semodule -i` against the package's own
  `cockpit.pp.bz2` loads it.

**⚠️ SELinux Disabled→Enforcing transition warning:** if a board's
`/etc/selinux/config` was never written, simply setting
`SELINUX=enforcing` and touching `/.autorelabel` before rebooting is
**not safe** — the real rootfs relabel can silently fail to run before
Enforcing mode engages, causing PID 1 to freeze solid on every device
node's SELinux context (`Failed to allocate manager object`, permanent
hang requiring a power cycle). The correct sequence: boot once with
`enforcing=0` on the kernel command line (a one-time systemd-boot menu
edit, not written to the boot config file), confirm the relabel actually
completed (`/.autorelabel` should be gone afterward), **then** set
`SELINUX=enforcing` for the next normal boot.

---

## Flash / Partition Layout

Raw offsets (written via `dd`, as the final build step, within the
fenced ranges below):

| Offset (sectors) | Content        | Notes |
|-------------------|----------------|-------|
| 64                 | idbloader.img  | DDR init blob + SPL |
| 16384              | u-boot.itb     | U-Boot proper + TF-A BL31 |

**SD card** (boot device — firmware + ESP only):

| Partition | Sector range | Size | FS | Mount | Content |
|-----------|--------------|------|----|----|---------|
| p1 SPL+   | 64 – 16383   | ~8 MB | — | — | fence over idbloader zone |
| p2 UBOOT  | 16384 – 24575 | 4 MB | — | — | fence over u-boot.itb zone |
| p3 RESERVED | 24576 – 32767 | 4 MB | — | — | unoccupied fence |
| p4 ESP    | 32768 – 4227071 | 2 GB | FAT32 | /efi | systemd-boot + kernel + initrd + DTB |
| p5 RootFS | 4227072 – (fixed, sized to NVMe's actual capacity) | sized to match NVMe | ext4 | — | unused on SD; present only for layout-template consistency |

**NVMe SSD** (root filesystem):

| Partition | FS | Mount | Content |
|-----------|----|----|---------|
| p1 (SPL/U-Boot fence equivalents) | — | — | present for layout consistency; permanently unreachable by the Boot ROM, inert |
| RootFS | ext4 | / | Fedora 44 aarch64 rootfs |

The NVMe's own SPL/U-Boot partition slots are inert by design — the Boot
ROM can never read from NVMe, so these partitions exist only to keep the
same partition-map template consistent across the fleet's boards, not
because anything is ever written to or read from them.

---

## Hardware Note — eMMC Not Used as a Boot Target

This board has an onboard eMMC module socket, normally a supported
Boot-ROM-tier target on RK3588 (same privileged tier as SD, per Orange
Pi's own documentation). On the specific eMMC module used with this
build, the SPL's DRAM/host-controller initialization against that module
fails at boot (error -95) — reproduced identically with both this
project's own SPL build and the unmodified vendor SPL, which rules out a
build-side cause and points to a hardware/BootROM-level SPL↔eMMC
interaction specific to this eMMC module. The eMMC hardware itself is not
faulty — the vendor's own older U-Boot can boot its factory OS from it
without issue; the failure is specific to the modern SPL's own init
sequence against this module. SD-boot + NVMe-root was chosen as the build
target instead, and is fully validated as described above.

---

## Recovery

With SD as the sole boot-capable device (eMMC not used, NVMe cannot be a
boot target at all on this SoC), there is no fallback boot path if the SD
card fails — unlike this lab's dual-boot-capable builds (e.g. Rock Pi 4's
SD/eMMC pair). Recovery from a corrupted SD requires re-flashing from a
known-good image; the NVMe's rootfs itself is unaffected by an SD
failure and can be re-paired with a freshly flashed SD carrying the same
root UUID reference.

---

Questions and corrections welcome — issues for specific problems,
discussions for questions and experience sharing.
