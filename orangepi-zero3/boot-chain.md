# Orange Pi Zero 3 — Fedora 43 aarch64 Boot Chain

A complete boot chain from SD card raw offsets to a running Fedora 43 aarch64
userspace on the Orange Pi Zero 3 (Allwinner H618). This documents what was
built, the problems encountered, and the current open items.

The goal: mainline U-Boot with EFI handoff, systemd-boot as boot manager,
a custom-built mainline kernel, Fedora 43 userspace — no vendor kernel, no
vendor bootloader, no Android blobs. The H618 presents a specific challenge:
the sunxi BootROM loads the SPL at a fixed raw offset (byte 8192, sector 16)
that overlaps the standard GPT partition entry array. The solution is to
relocate the GPT partition entry array past the end of the U-Boot binary using
`sgdisk -j`, keeping a clean GPT without MBR fallback.

**Hardware:** Orange Pi Zero 3, Allwinner H618 SoC (4× Cortex-A53 @ 1.5GHz,
aarch64), 4GB LPDDR4, 128GB SD card, Gigabit Ethernet (YT8511 PHY via EMAC).

---

## Fedora Build Toolkit

**Builder:** AArch64 builder VM (ARMbldr) — x86_64 host, aarch64 cross
toolchain (Fedora 43)

**Compiler packages:**
```
gcc-aarch64-linux-gnu    # 15.2.1 (Red Hat Cross)
```

**Build dependencies:**
```
bc  bison  flex  openssl-devel  elfutils-libelf-devel  perl  python3
make  dtc  u-boot-tools  swig  python3-setuptools  qemu-user-static
```

**Kernel:** Custom mainline kernel build — cross-compiled on ARMbldr.
No Fedora-packaged aarch64 kernel for H618 exists; mainline DTS support
for the OrangePi Zero 3 (`sun50i-h618-orangepi-zero3.dtb`) is present from
kernel 6.1 onward. This is a packaging gap, not a mainline gap — when Fedora's
aarch64 kernel package includes the H618 DTB, this board should migrate to the
packaged kernel path.

**Rootfs bootstrap:** Fedora 43 minimal rootfs assembled via
`dnf --installroot --forcearch=aarch64` on ARMbldr. Kernel modules installed
separately via tar into the rootfs.

**Initramfs:** Built with dracut inside a QEMU aarch64 chroot on ARMbldr.

---

## Source Versions

| Component    | Version        | Notes / Key Config                                                      | Source                        |
|--------------|----------------|-------------------------------------------------------------------------|-------------------------------|
| TF-A BL31    | 2.12.0         | `PLAT=sun50i_h616`, embedded in U-Boot FIT                              | git.trustedfirmware.org/TF-A  |
| U-Boot       | 2026.04        | `orangepi_zero3_defconfig`, `CONFIG_SYS_MMCSD_RAW_MODE_U_BOOT_SECTOR=0x50` | github.com/u-boot/u-boot  |
| Linux        | 7.0.0-rc6      | `sun50i-h618-orangepi-zero3.dtb`, custom defconfig                      | git.kernel.org                |
| systemd-boot | 258.7          | Installed via `bootctl install` in QEMU aarch64 chroot                  | Fedora 43 systemd package     |
| Fedora       | 43             | aarch64, minimal rootfs via `dnf --installroot`                         | Fedora repos                  |

---

## The Boot Chain

```
H618 Boot ROM
  → U-Boot SPL                      (SD card, byte offset 8192 / sector 16, raw)
  → TF-A BL31 v2.12.0              (embedded in U-Boot FIT at sector 0x50)
  → U-Boot proper v2026.04          (SD card, FIT image at sector 0x50)
  → systemd-boot 258.7              (SD card ESP, FAT32, BOOTAA64.EFI)
  → Linux 7.0.0-rc6 aarch64        (SD card ESP, BLS entry with initramfs + DTB)
  → Fedora 43 aarch64              (SD card rootfs, ext4)
```

---

## Stage 1 — H618 Boot ROM

The H618 Boot ROM is mask ROM — immutable, runs immediately after power-on.
It locates the SPL at a hardcoded raw byte offset of **8192 (sector 16)** on
the SD card. It produces no UART output; silence after power-on means the Boot
ROM did not find a valid SPL header.

**The GPT conflict:** Standard GPT places the partition entry array at sectors
2–33 (bytes 1024–17407). The U-Boot binary written at sector 16 overwrites
sectors 16 through ~2199 (for a ~1.1MB binary), destroying the primary GPT
partition entry array on every write. The solution is to **relocate the entry
array** past the end of U-Boot using `sgdisk -j 2210` before creating
partitions. The backup GPT at the end of the disk is unaffected, and the
primary GPT survives all subsequent U-Boot writes intact.

---

## Stage 2 — U-Boot SPL (SD card sector 16)

The SPL initializes DRAM (4096MB LPDDR4), then loads the U-Boot FIT image from
the SD card. The FIT image offset is set by
`CONFIG_SYS_MMCSD_RAW_MODE_U_BOOT_SECTOR`. The default of `0x60` did not match
the actual FIT location in this build; confirmed at sector `0x50` (byte offset
40960) by inspecting the binary for the FIT magic bytes (`d00dfeed`).

Three config overrides must be applied after `make defconfig` — they are not
in the defconfig defaults:

```
CONFIG_SPL_SYS_MALLOC_F_LEN=0x10000
CONFIG_SPL_STACK_R_MALLOC_SIMPLE_LEN=0x200000
CONFIG_SYS_MMCSD_RAW_MODE_U_BOOT_SECTOR=0x50
```

Without `SPL_SYS_MALLOC_F_LEN` and `SPL_STACK_R_MALLOC_SIMPLE_LEN`, the SPL
fails with "alloc space exhausted" / "Could not get FIT buffer" before loading
U-Boot proper.

**Build:**
```bash
make CROSS_COMPILE=aarch64-linux-gnu- ARCH=arm orangepi_zero3_defconfig
# Apply the three config overrides above (scripts/config or menuconfig)
make CROSS_COMPILE=aarch64-linux-gnu- ARCH=arm BL31=<path/to/bl31.elf>
# Produces u-boot-sunxi-with-spl.bin
```

**Flash** (raw offset — always write LAST, after all partition operations):
```bash
dd if=u-boot-sunxi-with-spl.bin of=/dev/sdX bs=1024 seek=8 conv=notrunc
```

**Critical:** Any `sgdisk`, `mkfs`, or `partprobe` operation after this `dd`
will overwrite the SPL at sector 16. Write U-Boot last, every time.

---

## Stage 3 — TF-A BL31 v2.12.0 (embedded in U-Boot FIT)

TF-A BL31 runs at EL3, providing PSCI and the secure monitor. It is embedded
in the U-Boot FIT image — not a separate raw flash offset. The `sun50i_h616`
platform covers both H616 and H618.

```bash
make CROSS_COMPILE=aarch64-linux-gnu- PLAT=sun50i_h616 bl31
# Produces build/sun50i_h616/release/bl31.elf
```

---

## Stage 4 — U-Boot proper v2026.04

With DRAM initialized and BL31 running at EL3, U-Boot proper initializes
the peripheral stack and presents a UEFI environment. It locates the ESP on
the SD card (mmcblk0p1) and hands off to `EFI/BOOT/BOOTAA64.EFI` via standard
`bootefi`.

U-Boot identifies the board as "OrangePi Zero3" and correctly reports DRAM as
4096MB. The MAC address is derived from the hardware serial number — unique per
board, no static assignment needed.

---

## Stage 5 — systemd-boot 258.7

systemd-boot is installed as `EFI/BOOT/BOOTAA64.EFI` on the FAT32 ESP via
`bootctl install` run inside a QEMU aarch64 chroot on the builder VM.

`efivarfs` is mounted read-only at runtime because U-Boot was built without
`CONFIG_EFI_RT_VOLATILE_STORE=y`. As a result, `bootctl` reports "not installed
in ESP" and "No boot loaders listed in EFI Variables" — cosmetic only, the boot
chain is fully functional. Rebuilding U-Boot with `CONFIG_EFI_RT_VOLATILE_STORE=y`
(as used on the Rock Pi 4 Plus) will make `efivarfs` writable and allow full
`bootctl` registration.

**BLS entry** (`/boot/loader/entries/<machine-id>-7.0.0-rc6.conf`):
```
title      Fedora Linux (7.0.0-rc6) on OrangePi Zero 3
linux      /EFI/fedora/<machine-id>/7.0.0-rc6/linux
initrd     /EFI/fedora/<machine-id>/7.0.0-rc6/initrd
devicetree /EFI/fedora/<machine-id>/7.0.0-rc6/dtb/allwinner/sun50i-h618-orangepi-zero3.dtb
options    root=/dev/mmcblk0p2 rw rootwait console=ttyS0,115200
```

`root=/dev/mmcblk0p2` is used rather than UUID — a future improvement once
a stable packaged kernel path is in place.

---

## Stage 6 — Linux 7.0.0-rc6 aarch64 (custom build)

The kernel is cross-compiled on ARMbldr targeting aarch64. The H618 DTS
(`sun50i-h618-orangepi-zero3.dtb`) is present in mainline from kernel 6.1
onward. The DTB is passed via the systemd-boot BLS `devicetree` field.

```bash
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- defconfig
# Enable H618/sunxi drivers; disable unneeded subsystems
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- Image dtbs modules
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- \
     INSTALL_MOD_PATH=<rootfs> modules_install
```

The initramfs is built with dracut inside a QEMU aarch64 chroot:
```bash
dracut --force --kver 7.0.0-rc6
```

Benign boot messages (not errors):
- `KASLR disabled due to lack of seed` — U-Boot does not seed the RNG; harmless
- RTC showing 1970 — no battery-backed RTC; chrony corrects on first NTP sync
- `dw-apb-uart / sun6i-spi: Error applying setting` — pinctrl quirk at init, cosmetic
- `panfrost 1800000.gpu: probe failed -110` — H618 GPU not fully supported; unused on server
- `dwmac-sun8i: Failed to restore VLANs` — no VLANs configured; ethernet fully functional

```
[root@opiz3 ~]# uname -a
Linux opiz3 7.0.0-rc6 #1 SMP PREEMPT ... aarch64 GNU/Linux

[root@opiz3 ~]# cat /etc/fedora-release
Fedora release 43 (Forty Three)
```

---

## Stage 7 — Fedora 43 aarch64

Fedora 43 aarch64 minimal userspace. Rootfs on SD card (mmcblk0p2, ext4,
label `root`). ESP on mmcblk0p1 (FAT32, label `ESP`). Both defined in
`/etc/fstab` using `LABEL=` entries. NetworkManager, sshd, chrony, and
Cockpit running. Ethernet functional via the EMAC/YT8511 PHY (interface `end0`).

**Post-install notes for a minimal rootfs via `dnf --installroot`:**
- Run `update-ca-trust extract` before first `dnf` — the CA bundle is absent
  from the minimal tarball
- Bootstrap the clock with `date -s` if starting from 1970 — TLS cert
  validation fails on a 1970 timestamp
- The `filesystem` RPM conflicts with `/lib64` as a real directory (usr-merge
  layout from tarball extraction). A full `dnf distrosync` resolves it;
  individual package installs succeed despite the transaction warning
- `update-crypto-policies --set DEFAULT` needed before Cockpit TLS works
- Mask `systemd-zram-setup@zram0.service` and `zram-generator.service` —
  the zram generator fires on every boot and there is no zram config:
  ```
  systemctl mask systemd-zram-setup@zram0.service
  systemctl mask zram-generator.service
  ```
  Note: masking `dev-zram0.device` alone is insufficient — the generator
  instantiates `systemd-zram-setup@zram0.service` directly at runtime
- `systemd-boot-unsigned` must be explicitly installed if `bootctl update`
  is needed — it is not pulled in by default in a minimal rootfs

---

## SD Card Partition Layout

GPT with relocated partition entry array (`sgdisk -j 2210`):

| Sectors       | Content                      | Notes                                       |
|---------------|------------------------------|---------------------------------------------|
| 0–1           | Protective MBR + GPT header  | Standard                                    |
| 16–2199       | U-Boot SPL + FIT             | Raw, written last (~1.1MB binary)           |
| 2210–2241     | GPT partition entry array    | Relocated past U-Boot end — survives write  |
| 4096–1052671  | ESP (mmcblk0p1, 512MB FAT32) | systemd-boot + kernel Image + DTB + initrd  |
| 1052672–end   | root (mmcblk0p2, ext4)       | Fedora 43 rootfs + kernel modules           |

**Partition setup:**
```bash
sgdisk -o /dev/sdX
sgdisk -j 2210 /dev/sdX
sgdisk -n 1:4096:+512M  -t 1:EF00 -c 1:"EFI System"       /dev/sdX
sgdisk -n 2:0:0         -t 2:8300 -c 2:"Linux filesystem"  /dev/sdX
mkfs.fat -F32 -n ESP  /dev/sdX1
mkfs.ext4 -L root     /dev/sdX2
# ... populate ESP and rootfs ...
dd if=u-boot-sunxi-with-spl.bin of=/dev/sdX bs=1024 seek=8 conv=notrunc
```

---

## Recovery

If the primary GPT is corrupted, the backup GPT at the end of the disk is
intact. From a Linux host:

```bash
sgdisk -e /dev/sdX          # rebuild primary GPT from backup
sgdisk --rebuild-crc /dev/sdX
```

The U-Boot prompt is reachable via serial (ttyS0, 115200 8N1) if SPL loads
but the boot chain fails. Interrupt with any key during the countdown.

---

## Open Items

- **SPI NOR boot** — vendor bootloader erased; our U-Boot not yet flashed to
  SPI NOR. Board currently boots from SD card only. Flash with `flashcp` (not
  `dd`) — SPI NOR requires erase before write.
- **`bootctl update` / writable efivarfs** — rebuild U-Boot with
  `CONFIG_EFI_RT_VOLATILE_STORE=y` to make efivarfs writable, then run
  `bootctl install` on the board. `systemd-boot-unsigned` RPM also required
  (`dnf install systemd-boot-unsigned`).
- **`dnf distrosync`** — `filesystem` RPM conflict (real `/lib64` dir vs
  symlink) blocks some transactions. Full distrosync resolves it.
- **Kernel/userspace alignment** — running 7.0.0-rc6 against Fedora 43
  userspace (packaged for 6.19.x). Functional for server use. Resolve at F44
  rebuild: check whether the Fedora 44 aarch64 kernel package includes the H618
  DTB; if so, migrate to the packaged kernel path.

---

Questions and corrections welcome — issues for specific problems,
discussions for questions and experience sharing.
