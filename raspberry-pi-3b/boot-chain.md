# Raspberry Pi 3 Model B — Fedora 43 aarch64 Boot Chain

A complete boot chain from VideoCore GPU firmware to a running Fedora 43
aarch64 userspace on the Raspberry Pi 3 Model B (Broadcom BCM2837).
This documents what was built, the problems encountered, and the solutions
found.

The goal: U-Boot with EFI handoff, systemd-boot as boot manager, Fedora
packaged kernel with dracut initramfs, `dnf update kernel` working without
manual intervention. The BCM2837 is ARMv8-A (64-bit) hardware that Raspbian
runs in 32-bit mode by convention — this build runs it in native aarch64 mode.

**Hardware:** Raspberry Pi 3 Model B Rev 1.2 (`a02082`), Broadcom BCM2837
SoC (quad-core Cortex-A53, ARMv8-A, 1.2GHz), 1GB LPDDR2, microSD storage,
10/100 USB-Ethernet (LAN9514/smsc95xx), BCM43438 WiFi/BT.

---

## Build Toolkit

**Kernel:** Fedora-packaged aarch64 kernel — installed via `dnf --installroot
--forcearch=aarch64`. No kernel source build required. Install both
`kernel-core` and `kernel-modules` — `kernel-core` alone omits the BCM2835
peripheral drivers needed for SD card access.

**Initramfs:** Built with `dracut` in an aarch64 chroot. Must include the
BCM2835 SD card driver stack explicitly — see dracut config below.

**systemd-boot:** Must be obtained from the Fedora aarch64
`systemd-boot-unsigned` RPM — `bootctl install` on an x86_64 host installs
`BOOTX64.EFI` which U-Boot will not load. Extract `systemd-bootaa64.efi`
from the aarch64 RPM directly.

---

## Source Versions

| Component    | Version                    | Notes                                                    | Source                          |
|--------------|----------------------------|----------------------------------------------------------|---------------------------------|
| GPU firmware | 2025-04-30T13:35:18        | Proprietary Broadcom blob — unavoidable on this platform | github.com/raspberrypi/firmware |
| U-Boot       | 2026.01                    | rpi_3_b_plus_defconfig, EFI_LOADER=y, EFI_VARIABLE_FILE_STORE=y, EFI_RT_VOLATILE_STORE=y | github.com/u-boot/u-boot |
| Linux        | 6.19.11-200.fc43.aarch64   | Fedora-packaged aarch64 kernel + kernel-modules          | Fedora 43 repos                 |
| systemd-boot | 258-1.fc43                 | BOOTAA64.EFI — from aarch64 RPM                         | Fedora 43 aarch64 repos         |
| Fedora       | 43                         | aarch64                                                  | Fedora repos                    |

---

## The Boot Chain

```
SoC power-on
  → VideoCore GPU ROM (immutable, in BCM2837 silicon)
      Reads bootcode.bin from FAT partition (GPU stage 1)
      Reads start.elf + fixup.dat from FAT partition (GPU stage 2)
      Reads config.txt — applies overlays, sets arm_64bit=1
      Loads u-boot.bin into RAM as the "kernel"
      Loads bcm2837-rpi-3-b.dtb, applies dtoverlays
      Releases ARM CPU reset
  → U-Boot 2026.01 (aarch64)
      Initialises hardware, enumerates MMC controllers
      Presents UEFI 2.11 EFI environment to next stage
      Locates systemd-boot EFI binary
  → systemd-boot 258 (BOOTAA64.EFI)
      Reads loader/loader.conf and BLS entries from FAT partition
      Displays boot menu (5s timeout)
      Loads kernel (vmlinuz), initramfs, and DTB per BLS entry
      Passes bootargs from BLS options= field
  → Linux 6.19.11-200.fc43.aarch64
      Initialises hardware via DTB
      bcm2835-dma loads (required by sdhost)
      bcm2835 sdhost driver binds to mmc@7e202000 → mmcblk0
      mmc_block creates mmcblk0p1/p2 block devices
      dracut initramfs mounts mmcblk0p2 as rootfs by UUID
  → Fedora 43 aarch64 userspace
      systemd PID 1
      NetworkManager, sshd, chronyd
```

---

## Stage Annotations

### VideoCore GPU ROM
The BCM2837 boots from a ROM embedded in the VideoCore GPU, not the ARM
CPU. The ARM cores are held in reset while the GPU executes the boot
sequence. On most modern SBCs the CPU executes code from the first moment
of power-on — the RPi is unusual in that the GPU is entirely in charge
until it hands off to U-Boot.

The GPU firmware (`start.elf`) is a proprietary Broadcom blob. It cannot
be replaced or bypassed. It reads `config.txt` and applies device tree
overlays *before* passing the DTB to U-Boot. This means the DTB that
U-Boot sees reflects the GPU firmware's interpretation of `config.txt`,
not just the raw DTB file on disk. `dtoverlay=disable-bt` is required
to get the GPU firmware to configure the sdhost controller correctly in
64-bit mode — without it, the sdhost node is not enabled in the passed DTB.

### Partition Table: GPT with Hybrid MBR
The BCM2837 GPU ROM reads sector 0 as an MBR partition table to locate
the FAT boot partition. Pure GPT produces a silently non-booting card —
no error, no output.

The solution is a **hybrid MBR**: GPT is the primary partition table,
with a legacy MBR partition 1 entry (type `0x0C`, FAT32 LBA) overlaid
using `gdisk`'s hybrid MBR feature. The GPU ROM finds the FAT partition
via the MBR entry; U-Boot and everything above it sees clean GPT with
correct EFI System Partition and architecture-specific root partition
type GUIDs.

**Build procedure:**
```bash
# Create GPT layout
sgdisk --new=1:0:+512M --typecode=1:EF00 --change-name=1:bootfs /dev/sdX
sgdisk --new=2:0:0     --typecode=2:B921B045 --change-name=2:rootfs /dev/sdX

# Overlay hybrid MBR — gdisk interactive
gdisk /dev/sdX
# r → h → 1 → n → 0c → n → n → w → Y
```

This gives the GPU ROM what it needs at sector 0 while preserving a
standards-compliant GPT for all layers above. Verified on RPi 3B Rev 1.2
with current GPU firmware (2026-04-13).

Note: RPi 4 and 5 have GPT-capable EEPROM firmware and do not require
this workaround.

### U-Boot 2026.01
Loaded by the GPU firmware as the "kernel" binary (`kernel=u-boot.bin` in
`config.txt`). U-Boot receives the GPU-modified DTB, initialises aarch64
hardware, and presents a UEFI 2.11 environment. EFI variables are stored
in a file on the FAT partition (`ubootefi.var`) — there is no dedicated
EFI variable storage on this platform. EFI vars are read-write
(`efivarfs` mounts rw).

### systemd-boot 258
Runs as a standard UEFI application. The `devicetree` field in BLS entries
is supported and used to pass the mainline kernel DTB
(`bcm2837-rpi-3-b.dtb`) explicitly. The boot menu appears on the serial
console at 115200 baud.

systemd-boot loads from `BOOTAA64.EFI` (fallback path) because U-Boot's
volatile EFI variable store does not persist boot entries across power
cycles. `bootctl status` reports all features operational.

### Linux 6.19.11-200.fc43.aarch64
Standard Fedora-packaged aarch64 kernel (`kernel-core` + `kernel-modules`).
The BCM2837 SD card controller (`mmc@7e202000`) uses the `bcm2835` sdhost
driver. Several modules in this stack are loadable (not built-in) in the
Fedora aarch64 kernel and must be explicitly forced into the dracut
initramfs:

| Module | Role | Notes |
|--------|------|-------|
| `bcm2835-dma` | DMA engine | Required by sdhost — silent failure without it |
| `bcm2835` | sdhost controller driver | Binds `mmc@7e202000` → `mmcblk0` |
| `mmc_core` | MMC subsystem core | Required by bcm2835 |
| `mmc_block` | MMC block device layer | Creates `mmcblk0p1/p2` — without it, no partition devices |

Without `mmc_block` in the initramfs, the kernel detects the card at the
MMC layer but never creates block device nodes — dracut waits indefinitely
for the by-uuid symlink that never appears. No error is logged.

The serial console is `ttyS0` (the BCM2835 auxiliary mini-UART at
`3f215040`). The PL011 UART (`3f201000`) is connected to the Bluetooth
module, not the GPIO header. `core_freq=250` in `config.txt` is required
to stabilise the mini-UART baud rate — the mini-UART clock derives from
the VPU clock which varies with CPU speed by default.

---

## Partition Layout

```
/dev/mmcblk0        GPT + hybrid MBR overlay on partition 1
├─mmcblk0p1  512M   FAT32  EF00 (ESP)          bootfs / ESP
│                          [MBR entry: 0x0C]   GPU ROM finds FAT here
│  bootcode.bin     GPU ROM stage 1 supplement
│  start.elf        GPU firmware
│  fixup.dat        GPU firmware companion
│  config.txt       GPU boot configuration
│  u-boot.bin       U-Boot aarch64 binary
│  bcm2837-rpi-3-b.dtb  Mainline DTB
│  EFI/BOOT/BOOTAA64.EFI
│  EFI/systemd/systemd-bootaa64.efi
│  loader/loader.conf
│  loader/entries/{machine-id}-{kver}.conf
│  {machine-id}/{kver}/linux
│  {machine-id}/{kver}/initrd
│  {machine-id}/{kver}/bcm2837-rpi-3-b.dtb
└─mmcblk0p2  rest   ext4   B921B045 (AArch64 root)   rootfs
```

---

## Configuration Files

**`config.txt` (complete working configuration):**
```ini
arm_64bit=1
kernel=u-boot.bin
device_tree=bcm2837-rpi-3-b.dtb
enable_uart=1
disable_fw_kms_setup=1
disable_overscan=1
core_freq=250
dtoverlay=disable-bt
```

**BLS entry options:**
```
root=UUID=<rootfs-uuid> rootfstype=ext4 rw
console=ttyS0,115200 8250.nr_uarts=1
earlycon=uart8250,mmio32,0x3f215040
systemd.machine_id=<machine-id>
```

**`/etc/kernel/install.conf`:**
```
BOOT_ROOT=/boot/efi
layout=bls
```

**`/etc/dracut.conf.d/bcm2835.conf`:**
```
force_drivers+=" bcm2835-dma bcm2835 mmc_core mmc_block "
```

---

## Serial Console

**Device:** ttyS0 (BCM2835 auxiliary mini-UART, `3f215040.serial`)  
**Baud:** 115200 8N1  
**GPIO header wiring (40-pin header):**
```
Pin 6  → GND
Pin 8  → TX (adapter RX)
Pin 10 → RX (adapter TX)
```

---

## Open Items

- **dnf update kernel** — kernel-install and dracut auto-rebuild not yet
  validated end-to-end on the target
- **efivarfs persistence** — EFI variables stored in `ubootefi.var` on FAT
  partition; persistence of `bootctl set-default` across reboots not verified
- **WiFi** — BCM43438 driver present but not configured

Questions and corrections welcome — issues for specific problems,
discussions for questions and experience sharing.
