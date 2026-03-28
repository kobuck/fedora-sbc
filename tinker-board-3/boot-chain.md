## ASUS Tinker Board 3 — Boot Chain with Annotations

Hardware: ASUS Tinker Board 3, Rockchip RK3566 SoC (AArch64, quad-core Cortex-A55),
4GB LPDDR4, SD card boot.

**Status: ⚠️ On hold — ethernet DMA reset failure under investigation**

---

### Note on Scope and Build History

This was the first build in this series. It predates the formalized scope
described in the repository overview — specifically, the goal of demonstrating
the boot chain to Fedora userspace **without** relying on dracut or an initramfs.

This build uses a dracut-generated initramfs, which is standard practice on
AArch64 and produces a working result. It is a departure from the scope these
examples now target: mainline kernel with drivers compiled in, no initrd, direct
boot to rootfs. The initrd approach is not wrong — it simply represents an
earlier path that predates the no-initrd objective.

The current hold status (ethernet DMA reset failure, see below) means the build
cannot be completed as-is regardless. When the ethernet issue is resolved, the
intent is to revisit this chain and align it with the current scope: no dracut,
drivers built in, same approach used in the JH7110 builds. What changes, what
becomes easier or harder on AArch64 with that approach, and what the tradeoffs
are will be documented at that point.

In the meantime this document captures what was built, what worked, and what
the current blocker is.

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
BLS entry and fstab. This is one of the practical differences from a no-initrd
build — UUID resolution happens in early userspace rather than requiring an
explicit device path in the kernel cmdline.

---

### Stage 6 — Linux kernel (AArch64)

Built from mainline linux using a custom DTS derived from the upstream
`rk3566-tinker-board-3.dtsi`. The build uses a dracut-generated initramfs,
which means storage drivers can be modules — the `=y` constraint that applies
to the JH7110 no-initrd builds does not apply here.

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
kernel-install + BLS when the initramfs path is in use — new kernels are added
to the ESP automatically. This is one of the areas that will need re-examination
if the build is revised to a no-initrd approach, since kernel-install behavior
with custom DTBs and built-in driver requirements differs.

---

### Known-Good Baseline

BSP kernel (6.6.89) + BSP DTB → boots to login, ethernet working.
Mainline kernel (6.19.8, 7.0-rc4) + mainline DTS → boots to login, ethernet DMA reset failure.

---

### Open Items

- **Ethernet DMA reset** — root cause under investigation, leads above untested
- **Build realignment** — when ethernet is resolved, revisit the build to align
  with current scope: no dracut, drivers built in, explicit device path for root
- **Upstream DTS contribution** — ethernet node additions to be submitted upstream
  once the DMA issue is resolved and the configuration is confirmed correct
