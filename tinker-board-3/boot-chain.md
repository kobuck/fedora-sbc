# ASUS Tinker Board 3 — Fedora 43 aarch64 Boot Chain

A complete boot chain from SD card raw offsets to a running Fedora 43 aarch64
userspace on the ASUS Tinker Board 3 (Rockchip RK3566). This documents what
was built, the problems encountered, and the current open items.

The goal: mainline U-Boot with EFI handoff, systemd-boot as boot manager,
current Fedora release, `dnf update kernel` working without manual
intervention. The Tinker Board 3 has no upstream DTS, the ethernet PHY
required a non-obvious DTS investigation to bring up, and the kernel is
currently the Rockchip BSP rather than mainline — all of this is documented
below.

**Hardware:** ASUS Tinker Board 3 R1.03, Rockchip RK3566 SoC (4x Cortex-A55,
aarch64), 4GB LPDDR4, Realtek RTL8211F-VD Gigabit Ethernet PHY, SD card
only (no onboard eMMC).

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

**Note:** `openssl-devel-engine`, `gnutls-devel`, and `python3-setuptools`
are consistently absent from a base Fedora install and must be added
explicitly. `arm-none-eabi-gcc` (bare-metal) is not available in Fedora
repos — `arm-linux-gnu-` works as a substitute for the TF-A M0 component.

**Rootfs bootstrap:** `dnf5 --installroot --forcearch=aarch64` on the builder VM.
Add `iproute` explicitly to the package list — it is absent from the default
bootstrap set and its absence breaks `ip` commands at first boot. After image
assembly, run `dnf reinstall glibc` in the chroot — glibc is not correctly
registered in the RPM database by the cross-arch install.

---

## Source Versions

| Component    | Version                    | Notes / Key Config                              | Source                             |
|--------------|----------------------------|-------------------------------------------------|------------------------------------|
| TF-A BL31    | rk3568_bl31_v1.45.elf      | Shared blob — RK3566 and RK3568 use same BL31  | github.com/rockchip-linux/rkbin    |
| U-Boot       | 2026.04                    | EFI_LOADER=y — base: rock-3c-rk3566_defconfig  | github.com/u-boot/u-boot           |
| Linux        | 6.6.89 (BSP develop-6.6)   | Rockchip BSP — custom DTS; dracut initramfs    | github.com/rockchip-linux/kernel   |
| systemd-boot | Fedora 43 systemd package  | bootctl install — full BLS conformance         | Fedora 43 systemd package          |
| Fedora       | 43                         | aarch64                                         | Fedora repos                       |

---

## The Boot Chain

```
RK3566 Boot ROM
  → idbloader.img                    (SD card, sector 64, raw offset)
  → TF-A BL31 rk3568_bl31_v1.45.elf (embedded in u-boot.itb)
  → U-Boot proper v2026.04           (SD card, sector 16384, raw offset)
  → (trust.img at sector 24576 — BL31 in u-boot.itb is authoritative;
     trust.img written separately per Rockchip convention)
  → systemd-boot                     (SD card ESP, FAT32)
  → Linux 6.6.89 + dracut initrd     (SD card ESP)
  → Fedora 43 aarch64               (SD card ext4 rootfs)
```

---

## Stage 1 — RK3566 Boot ROM

The RK3566 Boot ROM is mask ROM — immutable, runs immediately after power-on.
Like other Rockchip SoCs, it locates the bootloader by raw sector offset —
sector 64 must contain a valid idbloader signature. No partition type GUID
is required.

The Boot ROM produces no UART output. Absence of serial output after power-on
means the Boot ROM did not find a valid idbloader.

The Tinker Board 3 has no onboard eMMC and no SPI NOR. SD card is the sole
boot medium — there is no automatic fallback. Recovery requires reflashing the
SD card from another host.

---

## Stage 2 — idbloader (DDR init + SPL, sector 64)

`idbloader.img` is the Rockchip DDR initialization blob prepended to the
U-Boot SPL. The Boot ROM loads it from SRAM; it initializes the RK3566's
DDR controller, brings up 4GB of LPDDR4, then loads U-Boot proper from
sector 16384.

**U-Boot defconfig:** No Tinker Board 3 defconfig exists in mainline U-Boot.
`rock-3c-rk3566_defconfig` was chosen as the base — the ROCK 3C uses the
same Realtek RTL8211F PHY (`CONFIG_PHY_REALTEK=y`). A custom
`tinker-board-3-rk3566_defconfig` is saved in the U-Boot source tree.
U-Boot identifies the hardware as "Radxa ROCK 3C" — cosmetic, harmless.

**Build:**
```
make CROSS_COMPILE=aarch64-linux-gnu- ARCH=arm tinker-board-3-rk3566_defconfig
make CROSS_COMPILE=aarch64-linux-gnu- ARCH=arm BL31=<rk3568_bl31_v1.45.elf>
tools/mkimage -n rk3566 -T rksd -d spl/u-boot-spl.bin idbloader.img
```

**Flash** (raw offsets — always write after partitioning):
```
dd if=idbloader.img of=/dev/sdX seek=64    conv=notrunc
dd if=u-boot.itb    of=/dev/sdX seek=16384 conv=notrunc
dd if=trust.img     of=/dev/sdX seek=24576 conv=notrunc
```

---

## Stage 3 — TF-A BL31 (embedded in U-Boot FIT)

AArch64 requires a secure world firmware at EL3 to provide PSCI and a
trusted execution environment. TF-A BL31 initializes the secure monitor
and drops execution to EL2/EL1 for U-Boot proper and subsequently the OS.

The RK3566 and RK3568 share a BL31 binary — `rk3568_bl31_v1.45.elf` from
the Rockchip rkbin repository. This is a pre-built blob embedded in
`u-boot.itb` at compile time. A copy is also written at sector 24576
(`trust.img`) per Rockchip convention, but the embedded BL31 in the FIT
is what executes.

`arm-none-eabi-gcc` is not available in Fedora repos. Use `arm-linux-gnu-`
for the TF-A Cortex-M0 component (`M0_CROSS_COMPILE`).

---

## Stage 4 — U-Boot proper v2026.04

With DRAM up and TF-A BL31 running in EL3, U-Boot presents a UEFI environment.
It locates the ESP on the SD card (p1), finds `EFI/BOOT/BOOTAA64.EFI`, and
hands off via standard `bootefi`.

`CONFIG_EFI_LOADER=y` was already present in the rock-3c base defconfig —
no modification required. `CONFIG_EFI_RT_VOLATILE_STORE=y` added to enable
writable efivarfs and `bootctl install` EFI variable registration.

---

## Stage 5 — Linux 6.6.89 (Rockchip BSP kernel)

**Why BSP, not mainline:** The Tinker Board 3 has no upstream DTS in the
mainline kernel. A mainline boot was attempted using a DTS decompiled from
the ASUS Debian image — it failed silently. BSP phandles are incompatible
with mainline driver bindings. The Rockchip BSP kernel (`develop-6.6` branch)
is the stable foundation; mainline migration is a future objective once a
conformant DTS is established.

**Why custom DTS:** No Tinker Board 3 DTS exists in either mainline or the
BSP kernel source. The DTS was extracted from a running ASUS Debian image
via `/sys/firmware/fdt`, decompiled, and committed to the build tree as
`rk3566-tinker-board-3.dts`.

**Ethernet DTS — two simultaneous bugs, both required to fix:**

1. **pinctrl M0 → M1:** The mainline DTSI referenced `gmac1m0_*` pinctrl
   groups. The board's MDIO and RGMII signals physically route through GPIO
   bank 4 (M1 mux), not bank 3. On M0 the PHY was completely unreachable —
   MDIO returning `0xffff` regardless of any other settings. Fix: switch all
   `gmac1m0_*` references to `gmac1m1_*` pinctrl groups.

2. **PHY reset GPIO:** `snps,reset-gpio` (deprecated MAC-level reset) was
   not deasserting in the 7.0-rc4 kernel. `reset-gpios` in the PHY node
   alone was also insufficient — phylib was not deasserting the GPIO during
   probe. Fix: add a `gpio-hog` node in the `gpio3` bank to unconditionally
   deassert the reset line at boot, before any driver probes:
   ```
   /* inside &gpio3 node */
   phy-reset-hog {
       gpio-hog;
       gpios = <RK_PC2 GPIO_ACTIVE_LOW>;
       output-low;   /* ACTIVE_LOW: output-low = physically HIGH = reset deasserted */
       line-name = "phy-reset-deassert";
   };
   ```

Both fixes must be present. Either one alone leaves the PHY unreachable.
The failure mode for the pinctrl issue — MDIO returning `0xffff` — gives
no indication that the signal routing is wrong. Check the schematic and
IO domain assignments before spending time on PHY register debugging.

**Build:**
```
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- rockchip_linux_defconfig
# apply DTS and menuconfig adjustments
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- Image dtbs modules
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- INSTALL_MOD_PATH=/tmp/modules modules_install
```

**Uses dracut initramfs:** UUID/LABEL syntax in BLS boot entries is safe.
A `kernel-install` plugin at `/etc/kernel/install.d/60-tinker-board-3-dtb.install`
injects the board DTB into the ESP and appends a `devicetree` line to the
BLS entry on every kernel install. The DTB permanent home on the board is
`/etc/kernel/dtbs/rk3566-tinker-board-3.dtb`.

---

## Stage 6 — Fedora 43 aarch64

Standard Fedora 43 aarch64 userspace bootstrapped via `dnf5 --installroot
--forcearch=aarch64` on the builder VM. Board boots to multi-user target
with zero failed units.

A fresh Fedora rootfs has `PermitRootLogin` disabled by default in sshd —
enable it explicitly in `sshd_config` if root SSH access is required.

```
[root@tinkr3 ~]# uname -a
Linux tinkr3 6.6.89-g1ba51b059f25-dirty #1 SMP ... aarch64 GNU/Linux

[root@tinkr3 ~]# cat /etc/fedora-release
Fedora release 43 (Forty Three)
```

---

## Flash / Partition Layout

SD card raw offsets (written with `dd` after partitioning):

| Offset (sectors) | Content    | Notes                                                     |
|------------------|------------|-----------------------------------------------------------|
| 64               | idbloader.img | DDR init + SPL                                         |
| 16384            | u-boot.itb    | U-Boot proper + TF-A BL31                              |
| 24576            | trust.img     | TF-A BL31 (also embedded in FIT — written per Rockchip convention) |

SD card partition table:

| Partition | Size      | FS    | Mount | Content                              |
|-----------|-----------|-------|-------|--------------------------------------|
| sdXp1     | 512MB     | FAT32 | /boot | ESP — systemd-boot + kernels + DTBs  |
| sdXp2     | remainder | ext4  | /     | Fedora 43 rootfs                     |

SD card is the sole boot medium — no eMMC, no SPI NOR.

---

## Recovery

The only recovery path on this board is reflashing the SD card from another
host. There is no eMMC, no SPI NOR, and no Boot ROM recovery mode.

```
dd if=tinker-board-3.img of=/dev/sdX bs=4M status=progress conv=fsync
```

A known-good image should be archived to NAS storage before making changes
to a working build.

---

## Open Items

- **Mainline kernel migration** — BSP kernel (6.6.89) is the current lock;
  a conformant DTS for submission upstream is a future objective
- **RGMII TX tuning** — link is up at 1Gbps; tx_delay/rx_delay values
  need final validation for full outbound frame reliability
- **Clean shutdown** — `reboot` and `shutdown` do not trigger a clean
  shutdown; power cycle required. Likely a DTS/PMIC gap.
- **cpufreq** — PMIC/regulator DTS mismatch prevents CPU frequency scaling;
  CPU runs at fixed frequency (non-critical for server use)
- **DTS upstream submission** — gmac1m1 pinctrl + gpio-hog corrections are
  candidates for upstream once the board has full mainline kernel support

---

Questions and corrections welcome — issues for specific problems,
discussions for questions and experience sharing.
