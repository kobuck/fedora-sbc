## Orange Pi RV — Boot Chain with Annotations

Hardware: Orange Pi RV, StarFive JH7110 SoC (RISC-V riscv64), 8GB RAM,
500GB NVMe (Crucial CT500P310SSD8), 128GB SD card for ESP and recovery.

---

### The Boot Chain

```
JH7110 Boot ROM
  → U-Boot SPL v2026.01       (SPI NOR, mtd0, 960K)
  → OpenSBI v1.8.1            (embedded in U-Boot FIT)
  → U-Boot proper v2026.01    (SPI NOR, mtd2, 15M)
  → systemd-boot              (NVMe ESP, FAT32)
  → Linux 6.19.0              (NVMe ESP, custom build)
  → Fedora 43 riscv64         (NVMe rootfs)
```

---

### Stage 1 — JH7110 Boot ROM

The JH7110 Boot ROM is mask ROM — burned at the factory, immutable. It runs
immediately after power-on. Its job is to locate a valid next-stage image in a
defined boot order and hand off.

For this board the Boot ROM checks SPI NOR first. If a valid U-Boot SPL is
present, it loads it into SRAM and jumps to it. If SPI NOR is empty or invalid,
it falls back to the SD card — the recovery path.

The Boot ROM produces no UART output and introduces no observable delay. If
U-Boot SPL output does not appear on the serial console within a second or two
of power-on, the Boot ROM did not find a valid image.

---

### Stage 2 — U-Boot SPL (SPI NOR, mtd0)

The SPL (Secondary Program Loader) is a minimal first-stage U-Boot that runs
from the SoC's on-chip SRAM — DRAM is not yet initialized. Its job is to
initialize the JH7110's DDR controller, bring up 8GB of LPDDR4, then load
U-Boot proper from SPI NOR into DRAM and jump to it.

Built from mainline U-Boot v2026.01 using `starfive_visionfive2_defconfig` as
the base. The Orange Pi RV shares the JH7110 SoC with the VisionFive 2 but
requires its own DTS and an EEPROM match patch. The SPL reads the board's
EEPROM serial string (`VF7110B1-2228-...`) to select the correct DTS at runtime.

---

### Stage 3 — OpenSBI v1.8.1 (embedded in U-Boot FIT)

RISC-V defines a Supervisor Binary Interface (SBI) — a firmware layer between
the hardware and the OS kernel that provides services (timer management,
inter-processor interrupts, system reset) without requiring the kernel to access
privileged hardware directly.

OpenSBI implements the SBI specification. It runs in M-mode (the highest RISC-V
privilege level) and drops the OS into S-mode (supervisor mode). The kernel
never runs in M-mode.

OpenSBI is not a separate flash partition on this board — it is packaged inside
the U-Boot FIT image as a firmware blob, built in at compile time.

---

### Stage 4 — U-Boot proper v2026.01

With DRAM up and OpenSBI initialized, U-Boot proper handles boot selection.

The current `bootcmd` uses a direct fatload+bootefi from the NVMe ESP:

```
nvme scan
fatload nvme 0:1 0x40200000 vmlinuz-6.19.0
fatload nvme 0:1 0x46000000 jh7110-orangepi-rv.dtb
bootefi 0x40200000 0x46000000
```

This is a workaround. The target state is U-Boot handing off to systemd-boot
via the standard EFI boot path, which then selects the kernel from BLS entries.
systemd-boot is present on the NVMe ESP and was functional at one point —
restoring it as the active EFI boot target is an open item. In practice the
difference is invisible to the running OS; it matters for `dnf update kernel`
automation.

The U-Boot environment persists in SPI NOR mtd1 (the 64K uboot-env partition).
`saveenv` at the U-Boot prompt writes there. Do not write to mtd1 directly.

---

### Stage 5 — Linux 6.19.0 (no initrd)

The JH7110's PCIe controller and clock domains require specific drivers to
initialize before the NVMe can be accessed. Without an initrd, any driver in
the path between power-on and rootfs mount must be compiled into the kernel
(`=y`), not built as a loadable module (`=m`). The critical set for NVMe boot
on JH7110:

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
mount with no useful diagnostics. This was the root cause of PCIe boot
failures in earlier sessions — not a kernel version problem, not a DTS
problem, a module vs built-in config problem.

The kernel is built from linux-stable v6.19 with a custom DTS for the Orange Pi
RV. The DTS is based on the upstream mainline DTS that landed after the v6.19
tag — a rebuild against v6.20 or later (when the DTS is native) should be
straightforward.

The rootfs uses an explicit device path (`root=/dev/nvme0n1p2`) rather than
UUID. Without initrd there is no early userspace to resolve UUID references —
UUID in the kernel cmdline causes an immediate panic.

---

### Stage 6 — Fedora 43 riscv64

Standard Fedora 43 riscv64 userspace. systemd, NetworkManager, DNF, and SSH
all function normally. The board survived a full `dnf upgrade` and reboot with
no intervention.

```
[root@opirv ~]# uname -a
Linux opirv 6.19.0-dirty #7 SMP Fri Mar 14 ...  riscv64 GNU/Linux

[root@opirv ~]# cat /etc/fedora-release
Fedora release 43 (Forty Three)

[root@opirv ~]# dnf upgrade --refresh
[runs cleanly, packages update normally]
```

---

### Recovery

Insert the SD card. U-Boot's boot priority is SD card over NVMe — software
policy, no physical boot switch. A valid SPL and bootable ESP on the SD card
will boot the board regardless of NVMe state.

SPI NOR can be recovered via xmodem serial if the SPL is ever corrupted: hold
the BOOT button at power-on to enter Boot ROM recovery mode, then transfer a
new SPL image via the serial connection using `sx` from the `lrzsz` package.

---

### Open Items

- **systemd-boot restoration** — make it the active EFI boot target so U-Boot
  hands off via standard EFI path rather than direct fatload
- **kernel-install DTB plugin** — so `dnf update kernel` handles DTB injection
  automatically alongside the kernel image
- **fw_setenv / libubootenv** — not yet in the Fedora riscv64 repos; needed for
  managing U-Boot environment from userspace
- **WiFi** — AP6256 (BCM43456) on SDIO, not yet enabled in DTS
