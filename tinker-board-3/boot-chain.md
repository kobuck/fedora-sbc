## ASUS Tinker Board 3 — Boot Chain with Annotations

Hardware: ASUS Tinker Board 3, Rockchip RK3566 SoC (AArch64, quad-core Cortex-A55),
4GB LPDDR4, SD card boot.

**Status: ⚠️ On hold — ethernet DMA reset failure under investigation**

---

### AArch64 and the Initramfs Path

On AArch64, this build uses a Fedora-packaged kernel installed via `dnf` and a
dracut-generated initramfs. This is the correct and intended path for AArch64
boards — dracut is a Fedora package, kernel-install integrates with BLS, and
`dnf update kernel` works automatically without manual intervention.

The no-initrd approach used in the JH7110 builds (Milk-V Mars, Orange Pi RV)
is a workaround for a packaging gap: Fedora does not ship riscv64 kernel RPMs
in standard repos, so a custom kernel build is required. Without an initramfs,
all rootfs-path drivers must be built in (`=y`). That constraint is specific to
riscv64 and does not apply here.

dracut on AArch64 is not a compromise — it is the policy-correct path.

---

### The Boot Chain

```
RK3566 Boot ROM
  → U-Boot SPL / idbloader     (SD card raw offset, AArch64)
  → TF-A BL31                  (ARM Trusted Firmware, EL3)
  → U-Boot proper v2024.x      (SD card raw offset)
  → systemd-boot               (SD card ESP, FAT32)
  → Linux 7.0-rc4 / 6.19.x    (SD card ESP)
  → Fedora 43 aarch64          (SD card ext4 rootfs)
```

---

### Stage 1 — RK3566 Boot ROM

The RK3566 Boot ROM reads the first usable sectors of the SD card looking for
an idbloader header signature. On finding a valid idbloader, it loads it into
SRAM and executes it.

The Rockchip Boot ROM does not use partition type GUIDs — it looks for raw
header signatures at fixed offsets from the start of the storage medium. The
idbloader is written to sector 64 (32KB offset) on the SD card, outside any
partition table. This is the most significant structural difference from the
JH7110 boot sequence.

---

### Stage 2 — idbloader / U-Boot SPL (SD card, raw offset)

The idbloader packages the DDR initialization firmware and U-Boot SPL into a
single blob written to sector 64. It initializes LPDDR4, sets up the clock tree,
and loads TF-A and U-Boot proper into DRAM.

Built from mainline U-Boot using `rk3566-tinker-board-3` defconfig on a
cross-compile builder VM (x86_64 host, aarch64-linux-gnu- cross-compiler).
The Tinker Board 3 is a supported board in mainline U-Boot — no custom patches required.

---

### Stage 3 — TF-A BL31 (ARM Trusted Firmware)

TF-A BL31 is the AArch64 secure firmware layer, providing runtime services at
EL3 (the highest ARM exception level). It handles SMC (Secure Monitor Call)
requests from the OS for power management, PSCI (power state coordination), and
related services. This is the AArch64 architectural equivalent of OpenSBI on
RISC-V — both are the secure firmware layer that the OS kernel trusts but never
directly controls.

On RK3566, TF-A is packaged inside the idbloader or U-Boot FIT — not a separate
flash partition. After BL31 initializes, execution drops to EL2/EL1 where
U-Boot proper runs.

---

### Stage 4 — U-Boot proper

Standard U-Boot EFI boot — scans the SD card ESP for `EFI/BOOT/BOOTAA64.EFI`
(systemd-boot) and hands off via `bootefi`. Built with `CONFIG_EFI_LOADER=y`.

U-Boot environment stored on the SD card in a dedicated env partition.

---

### Stage 5 — systemd-boot

systemd-boot reads BLS entries from `loader/entries/`, loads the selected kernel
image and DTB, and boots via EFI handoff.

With a dracut initramfs, UUID-based root references work correctly in both the
BLS entry and fstab — UUID resolution happens in early userspace, before rootfs
mount.

---

### Stage 6 — Linux kernel (AArch64)

Built from mainline linux using a custom DTS derived from the upstream
`rk3566-tinker-board-3.dtsi`. The build uses a Fedora-packaged kernel and a
dracut-generated initramfs — storage drivers can be modules and load normally
via early userspace.

**Current hold status:** Ethernet (gmac1, RTL8211F) fails with DMA reset
errors under both kernel 6.19.8 and 7.0-rc4. Investigation to date:

- pinctrl gmac1m0 confirmed correct against schematic
- RGMII delay values (0x36/0x2b) verified
- CLK_MAC1_2TOP, aclk/pclk_gmac1 enabled at correct frequencies
- vccio7 at 3.3V confirmed
- All known DTS suspects cleared

The board boots cleanly to login with networking disabled. Ethernet TX+RX
works on the BSP vendor kernel (6.6.89), confirming the hardware is functional.
The DMA reset failure is a mainline kernel interaction with this board's
configuration — root cause not yet identified.

Question posted to `#linux-rockchip` on Libera IRC. Awaiting response.

New leads received (2026-03-27):
1. `snps,reset-delays-us` — stmmac reset timing changed between 6.8 and 6.12;
   explicit delay values may be required in the DTS
2. Diff of BSP vs mainline stmmac driver for reset/clock patches not yet upstreamed
3. MAC `2e:55` is locally-administered — verify whether BSP sets MAC explicitly
   vs mainline deriving it from fuse

These have not yet been tested. They represent the current investigation path.

---

### Stage 7 — Fedora 43 aarch64

Standard Fedora 43 aarch64 rootfs. `dnf update kernel` works correctly via
kernel-install + BLS — new kernels are added to the ESP automatically.

---

### Known-Good Baseline

BSP kernel (6.6.89) + BSP DTB → boots to login, ethernet working.
Mainline kernel (6.19.8, 7.0-rc4) + mainline DTS → boots to login, ethernet DMA reset failure.

---

### Open Items

- **Ethernet DMA reset** — root cause under investigation, leads above untested
- **Upstream DTS contribution** — ethernet node additions to be submitted upstream
  once the DMA issue is resolved and the configuration is confirmed correct
