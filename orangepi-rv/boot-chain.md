## Orange Pi RV — Full Boot Chain with Annotations

Following up on the tinkr3 post with the boot chain for my second SBC building:
the Orange Pi RV, running Fedora 43 riscv64. This one was considerably more
involved than the AArch64 build — RISC-V on this hardware has some sharp edges —
but the result is a clean, fully autonomous boot from SPI NOR through to a
standard Fedora userspace over NVMe.

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

The JH7110 Boot ROM is the first code that runs after power-on. It is
mask ROM — burned at the factory, immutable. Its job is narrow: look for
a valid next-stage image in a defined boot order and hand off to it.

For this board the Boot ROM checks SPI NOR first. If it finds a valid
U-Boot SPL there, it loads it into SRAM and jumps to it. If SPI NOR is
empty or invalid, it falls back to the SD card. This fallback behavior
is the recovery path — insert an SD card with a valid SPL and the board
will boot regardless of what is in SPI NOR.

The Boot ROM does almost nothing visible — no UART output, no delay.
If U-Boot SPL doesn't start appearing on the serial console within a
second or two of power-on, the Boot ROM didn't find a valid image.

---

### Stage 2 — U-Boot SPL (SPI NOR, mtd0)

SPL stands for Secondary Program Loader — it is a minimal first-stage
U-Boot whose only job is to initialize DRAM and load the full U-Boot
image from SPI NOR into RAM.

The JH7110 Boot ROM loads the SPL into a small SRAM buffer (the SoC's
on-chip SRAM, not DRAM — DRAM isn't initialized yet). The SPL runs from
there, initializes the JH7110's DDR controller to building the 8GB LPDDR4,
then loads U-Boot proper from SPI NOR mtd2 into DRAM and jumps to it.

This is built from mainline U-Boot v2026.01 using
`starfive_visionfive2_defconfig` as the base. The Orange Pi RV is closely
related to the VisionFive 2 hardware — same SoC, similar board layout —
but requires its own DTS and an EEPROM match patch. The SPL reads the
board's EEPROM serial string (`VF7110B1-2228-...`) to select the correct
DTS at runtime.

---

### Stage 3 — OpenSBI v1.8.1 (embedded in U-Boot FIT)

This stage has no direct equivalent on AArch64. RISC-V defines a Supervisor
Binary Interface (SBI) — a firmware layer that sits between the hardware
and the OS kernel, providing services the kernel needs (timer, IPI, system
reset) without requiring the kernel to access privileged hardware directly.

OpenSBI implements the SBI specification. It runs in M-mode (machine mode,
the highest RISC-V privilege level) and drops the OS into S-mode (supervisor
mode) where it runs for the rest of the session. The OS kernel never runs
in M-mode.

On this board OpenSBI is not a separate flash partition — it is packaged
inside the U-Boot FIT image (Flattened Image Tree) as a firmware blob.
U-Boot loads OpenSBI, OpenSBI initializes, then U-Boot proper gets control
back to handle boot selection.

If you've worked with TF-A (Trusted Firmware-A) on AArch64, OpenSBI
occupies a similar architectural position — it's the secure firmware
layer that the OS kernel trusts but never directly controls.

---

### Stage 4 — U-Boot proper v2026.01

With DRAM up, OpenSBI initialized, and execution back in its hands,
U-Boot proper handles the actual boot selection. It scans for bootable
media, reads its boot script or environment, and hands off to the next
stage.

The current `bootcmd` does a direct fatload+bootefi from the NVMe ESP:

```
nvme scan
fatload nvme 0:1 0x40200000 vmlinuz-6.19.0
fatload nvme 0:1 0x46000000 jh7110-orangepi-rv.dtb
bootefi 0x40200000 0x46000000
```

This is a temporary workaround. The target state is U-Boot handing off
to systemd-boot via the standard EFI boot path, which then selects the
kernel from BLS entries. systemd-boot is present on the NVMe ESP and
was functional at one point — restoring it as the active EFI boot target
is an open item. In practice the difference is invisible to the running
OS; it matters for `dnf update kernel` automation.

The U-Boot environment is saved to SPI NOR mtd1 (the 64K uboot-env
partition). Changes made with `saveenv` persist across reboots without
touching mtd0 or mtd2.

---

### Stage 5 — Linux 6.19.0 (custom build, no initrd)

This is where RISC-V on this hardware diverges most sharply from a
standard Fedora kernel install.

The JH7110's PCIe controller and clock domains require specific drivers
to initialize before the NVMe can be accessed. On most systems these
would be kernel modules loaded by an initrd (initial RAM disk) early in
the boot sequence. But on this board there is no initrd — the kernel
boots directly to the rootfs on NVMe without an early userspace stage.

This means any driver in the path between power-on and rootfs mount must
be compiled into the kernel (`=y`), not built as a loadable module (`=m`).
The critical ones for NVMe boot on JH7110:

```
CONFIG_PCIE_STARFIVE_HOST=y
CONFIG_CLK_STARFIVE_JH7110_STG=y
CONFIG_CLK_STARFIVE_JH7110_AON=y
CONFIG_BLK_DEV_NVME=y
CONFIG_NVME_CORE=y
CONFIG_PHY_STARFIVE_JH7110_PCIE=y
```

Getting this wrong produces a silent failure: the PCIe PHY doesn't
initialize, the NVMe is never enumerated, and the kernel panics at
rootfs mount. The fix is a full kernel rebuild with the correct config.
This was the root cause of the PCIe issues I hit in earlier build
sessions — not a kernel version problem, not a DTS problem, a module
vs built-in config problem.

The kernel is built from linux-stable v6.19 with a custom DTS for the
Orange Pi RV. The DTS is based on the upstream mainline DTS that landed
after the v6.19 tag — close enough that a rebuild against kernel 6.20
or later (when it carries the DTS natively) should be straightforward.

---

### Stage 6 — Fedora 43 riscv64

Once the kernel has the NVMe enumerated and the rootfs mounted, boot
continues into a standard Fedora 43 riscv64 userspace. systemd,
NetworkManager, DNF, SSH — all working normally.

```
[root@opirv ~]# uname -a
Linux opirv 6.19.0-dirty #7 SMP Fri Mar 14 ...  riscv64 GNU/Linux

[root@opirv ~]# cat /etc/fedora-release
Fedora release 43 (Forty Three)

[root@opirv ~]# dnf upgrade --refresh
[runs cleanly, packages update normally]
```

The board survived a full `dnf upgrade` and reboot this week with no
intervention. systemd-boot reported the same version already in place
after the systemd package upgraded — normal, the EFI binary was already
current.

---

### Recovery

The recovery path is simple: insert the SD card. U-Boot's boot priority
is TF card over NVMe — software policy, no physical boot switch. As long
as the SD card has a valid SPL and a bootable ESP, the board will boot
from it regardless of NVMe state.

SPI NOR can be recovered via xmodem serial if the SPL is ever corrupted:
hold the BOOT button while powering on to enter the Boot ROM's recovery
mode, then transfer a new SPL image via `/dev/ttyUSBx` on the host using
`sx` from the `lrzsz` package.

---

### What's Still Open

- **systemd-boot restoration** — make it the active EFI boot target
  so U-Boot hands off via standard EFI path rather than direct fatload
- **kernel-install DTB plugin** — so `dnf update kernel` can handle a
  custom DTB injection automatically
- **fw_setenv / libubootenv** — not yet in the Fedora riscv64 repos;
  needed for managing U-Boot environment from userspace
- **WiFi** — AP6256 (BCM43456) on SDIO, not yet enabled in DTS

Happy to answer questions on any stage. The hardest part by far was
the PCIe/NVMe building — if you're working on a JH7110 board and
hitting a boot hang after the kernel starts, the module vs built-in
config is the first thing to check.
