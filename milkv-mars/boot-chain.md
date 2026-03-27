## Milk-V Mars — Boot Chain with Annotations

This is a follow-on to my Orange Pi RV post. Same SoC (StarFive JH7110),
different board — the Milk-V Mars. The boot chain is closely related but
has some important differences, and the build is cleaner in a few respects:
systemd-boot is fully installed via `bootctl install`, EFI variables are
writable, and `dnf update kernel` should work correctly going forward.

Hardware: Milk-V Mars, StarFive JH7110 SoC (RISC-V riscv64), 8GB LPDDR4,
119GB SD card (no NVMe on this unit — M.2 E-Key slot unpopulated).

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

The Boot ROM is mask ROM — burned at the factory, immutable. It is the
first code that runs after power-on and its job is narrow: locate a valid
next-stage image and hand off.

The JH7110 Boot ROM has a configurable boot source selection controlled
by two GPIO pins (RGPIO_0 and RGPIO_1). On the Milk-V Mars these are
exposed as DIP switches on the board. The default factory setting is
SPI Flash (both switches off), which is what we use in production.

The Boot ROM locates bootloader partitions by **type GUID**, not by
partition number or position. The SPL partition must carry GUID
`2E54B353-1271-4842-806F-E436D6AF6985` and the U-Boot FIT partition
must carry `BC13C2FF-59E6-4262-A352-B275FD6F7172`. Using the default
sgdisk type code (`8300` Linux filesystem) causes the Boot ROM to skip
the partition entirely — no error message, the board appears dead.
This is one of the sharpest edges in JH7110 building.

If SPI NOR is corrupted or empty, flip the appropriate DIP switch to
SD card boot mode — the Boot ROM will look for SPL on the SD card
instead, which is the recovery path.

---

### Stage 2 — U-Boot SPL (SPI NOR, mtd0)

The SPL (Secondary Program Loader) is a minimal first-stage U-Boot.
It runs from the SoC's on-chip SRAM — DRAM is not yet initialized.
Its job is to initialize the JH7110's DDR controller, bring up 8GB of
LPDDR4, then load U-Boot proper from SPI NOR into DRAM and jump to it.

Built from mainline U-Boot v2026.01 using `starfive_visionfive2_defconfig`.
The Mars is native in mainline U-Boot — no patches, no custom DTS.
The SPL reads the board's EEPROM serial string (ours: `MARS-V11-...`) and
matches it against a 4-character prefix table to select the correct board
DTS at runtime. `MARS` matches the Milk-V Mars entry natively.

This stage produces brief output on the serial console (`U-Boot SPL 2026.01`)
then goes quiet while DRAM initializes, appearing after a second or two
with the full U-Boot banner.

---

### Stage 3 — OpenSBI v1.8.1 (embedded in U-Boot FIT)

RISC-V defines a Supervisor Binary Interface (SBI) — a firmware layer
between the hardware and the OS kernel providing services like timer
management, inter-processor interrupts, and system reset. OpenSBI
implements this specification.

OpenSBI runs in M-mode (machine mode, the highest RISC-V privilege level).
It initializes, prints its banner, then drops execution into S-mode
(supervisor mode) where U-Boot proper and subsequently the OS kernel run.
The kernel never accesses M-mode hardware directly — all such operations
go through SBI calls.

OpenSBI is not a separate flash partition on this board — it is packaged
inside the U-Boot FIT (Flattened Image Tree) image as a firmware blob,
built into the package at compile time. AArch64 readers will recognize
this role as analogous to TF-A (Trusted Firmware-A) BL31.

The OpenSBI banner in the boot log (`Platform Name: Milk-V Mars`) confirms
mainline platform support — the Mars has a full platform definition in
OpenSBI that sets up the correct HART topology, interrupt routing, and
power management for this board.

---

### Stage 4 — U-Boot proper v2026.01

With DRAM initialized, OpenSBI running in M-mode, and execution handed
back, U-Boot proper takes over boot selection.

U-Boot scans the SD card for a bootable ESP. It finds our FAT16 partition
(p3), locates `EFI/BOOT/BOOTRISCV64.EFI`, and hands off via the standard
UEFI boot path — `bootefi`. This is the key difference from the Orange Pi
RV build: U-Boot is using the proper EFI boot flow, not a manual `fatload`
workaround.

U-Boot on the Mars is built with `CONFIG_EFI_VARIABLE_FILE_STORE=y` and
`CONFIG_EFI_RT_VOLATILE_STORE=y`. The first stores EFI variables in a
file (`ubootefi.var`) on the ESP rather than NVRAM (which doesn't exist
on this SoC). The second makes efivarfs writable at runtime, so the OS
can create and modify EFI boot entries. Without `EFI_RT_VOLATILE_STORE`,
efivarfs mounts read-only and `bootctl install` cannot register a boot
entry — `bootctl` reports "systemd-boot not installed in ESP" even when
the binary is present.

The U-Boot environment (boot order, saved variables) persists in SPI NOR
mtd1 (a 64K uboot-env partition). `saveenv` at the U-Boot prompt writes
there. Never write to mtd1 directly with `flashcp` or `dd` — only through
`saveenv`.

---

### Stage 5 — systemd-boot 258.7

systemd-boot is a minimal UEFI boot manager. U-Boot hands it control via
`bootefi`, it reads `loader/loader.conf` and the BLS (Boot Loader
Specification) entries in `loader/entries/`, displays a menu (5 second
timeout), and loads the selected kernel.

On this build systemd-boot is fully installed via `bootctl install` —
not just placed at the fallback path. This means:

- The binary lives at the canonical path `EFI/systemd/systemd-bootriscv64.efi`
  in addition to the fallback `EFI/BOOT/BOOTRISCV64.EFI`
- A "Linux Boot Manager" EFI boot entry is registered in EFI Variables
  (stored in `ubootefi.var`)
- A random seed and system token are set for entropy

`bootctl status` shows the boot entry as active and in boot-order. This
is the conformant state — `dnf update kernel` will invoke `kernel-install`,
which will add new BLS entries to the ESP automatically.

---

### Stage 6 — Linux 6.19.0

The kernel is a custom build from linux-stable v6.19 using the mainline
`jh7110-milkv-mars.dts` (included in mainline since before v6.19). No
custom DTS, no patches — the Mars is fully supported upstream.

Unlike the Orange Pi RV build, the rootfs is on SD card (no NVMe), so the
critical built-in driver set is different. The SD card controller
(`MMC_DW_STARFIVE`) must be `=y` since there is no initrd. The boot
entry uses an explicit device path (`root=/dev/mmcblk1p4`) rather than
UUID — without initrd there is no early userspace to resolve UUID
references, and UUID in the kernel cmdline causes an immediate panic.

Two console entries appear in the kernel cmdline:
```
console=tty0 console=ttyS0,115200
```
This is intentional. Linux supports multiple simultaneous console
destinations — `tty0` sends kernel messages to the HDMI framebuffer,
`ttyS0` sends them to the serial UART. Both receive output. The last
`console=` entry is where interactive input is read, so `ttyS0` is
the interactive console. Standard practice for SBC work where you may
have both a monitor and a serial cable attached.

The kernel identifies the board correctly from the DTS:
```
Machine model: Milk-V Mars
```

---

### Stage 7 — Fedora 43 riscv64

Rootfs is a Fedora 43 riscv64 userspace bootstrapped using
`dnf5 --installroot --forcearch=riscv64` on a RISC-V builder VM
(RVbldr, x86_64 host running the riscv64 cross toolchain).

Cross-arch installroot with `tsflags=noscripts` requires several
post-install fixups that dnf5 cannot run in a foreign-arch chroot:
ldconfig, PAM configuration, usr-merge symlinks, crypto policy setup.
These are all scripted in a build script that produces a reproducible
image from scratch.

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

NetworkManager, sshd, and Cockpit (web management UI at port 9090) are
all running. The board survived a full boot cycle from this state with
no intervention.

---

### Mars vs Orange Pi RV — Key Differences

Both boards use the JH7110 SoC, so the boot chain is structurally
identical. The meaningful differences:

| | Milk-V Mars | Orange Pi RV |
|---|---|---|
| Rootfs device | SD card (mmcblk1p4) | NVMe (nvme0n1p2) |
| Critical built-in drivers | MMC_DW_STARFIVE | PCIe + NVMe stack |
| U-Boot board match | `MARS` EEPROM prefix | `VF7110B1` prefix + DTS patch |
| systemd-boot state | Fully installed via bootctl | Fallback path only |
| efivarfs | rw (EFI_RT_VOLATILE_STORE=y) | ro (missing config option) |
| DTS source | Mainline native (upstream) | Downstream, not yet merged |

The NVMe-vs-SD difference is the biggest practical one: PCIe on JH7110
requires a larger set of built-in drivers and is significantly harder
to get right. The Mars build was smoother in this respect — SD boot
is simpler and better supported in mainline.

---

### Recovery

**SD card direct boot:** Flip the DIP switch to SD card mode (GPIO0=1 on
the board's RGPIO pins). The Boot ROM will load SPL from the SD card
instead of SPI NOR. As long as the SD card has a valid SPL on its first
partition (with the correct type GUID), the board will boot.

**SPI NOR reflash:** With the board running (from SD or from a working
SPI NOR), load new binaries to the FAT ESP partition and use U-Boot's
`sf` commands at the U-Boot prompt:
```
sf probe
load mmc 1:3 $kernel_addr_r u-boot-spl.bin.normal.out
sf update $kernel_addr_r 0 $filesize
load mmc 1:3 $kernel_addr_r u-boot.itb
sf update $kernel_addr_r 0x100000 $filesize
```
**Never use `dd` directly to `/dev/mtdX`** for SPI NOR. NOR flash requires
an erase cycle before writing — raw `dd` skips this and corrupts the flash
silently. `sf update` handles erase+write correctly.

---

### What's Still Open

- **kernel-install DTB plugin** — so `dnf update kernel` handles DTB
  injection automatically alongside the kernel image
- **tio serial profile** — minor convenience: `tio milkv` instead of
  full device/baud specification
- **`fw_setenv` / libubootenv** — not yet in Fedora riscv64 repos;
  needed for managing U-Boot environment from userspace

Happy to answer questions. The JH7110 type-GUID requirement is the most
surprising thing to hit if you're coming from x86 or AArch64 — silent
failure with no diagnostics, easily mistaken for a hardware problem.
