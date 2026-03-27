## ASUS Tinker Board 3 — Boot Chain with Annotations

Hardware: ASUS Tinker Board 3, Rockchip RK3566 SoC (AArch64, quad-core Cortex-A55),
4GB LPDDR4, SD card boot.

**Status: ⚠️ On hold — ethernet DMA reset failure under investigation**

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

The RK3566 Boot ROM follows Rockchip's standard boot sequence. It reads the
first usable sectors of the SD card looking for an idbloader header signature.
On finding a valid idbloader, it loads it into SRAM and executes it.

Unlike JH7110 (StarFive), the Rockchip Boot ROM does not use partition type
GUIDs — it looks for raw header signatures at fixed offsets from the start of
the storage medium. The idbloader is written to sector 64 (32KB offset) on the
SD card, outside any partition table.

---

### Stage 2 — idbloader / U-Boot SPL (SD card, raw offset)

The idbloader packages the DDR initialization firmware and U-Boot SPL into a
single blob written to sector 64. It initializes LPDDR4, sets up the clock tree,
and loads TF-A and U-Boot proper into DRAM.

Built from mainline U-Boot using `rk3566-tinker-board-3` defconfig. The Tinker
Board 3 is a supported board in mainline U-Boot — no custom patches required.

---

### Stage 3 — TF-A BL31 (ARM Trusted Firmware)

TF-A BL31 is the AArch64 equivalent of OpenSBI on RISC-V — a secure firmware
layer that provides runtime services to the OS at EL3 (the highest ARM exception
level). It handles SMC (Secure Monitor Call) requests from the OS for power
management, PSCI (power state coordination), and related services.

On RK3566, TF-A is packaged inside the idbloader or U-Boot FIT — not a separate
flash partition. After BL31 initializes it drops execution to EL2/EL1 where
U-Boot proper runs.

---

### Stage 4 — U-Boot proper

Standard U-Boot EFI boot — scans the SD card ESP for `EFI/BOOT/BOOTAA64.EFI`
(systemd-boot) and hands off via `bootefi`. Built with `CONFIG_EFI_LOADER=y`.

U-Boot environment stored on the SD card in a dedicated env partition.

---

### Stage 5 — systemd-boot

Same as the JH7110 builds — reads BLS entries from `loader/entries/`, loads
the selected kernel image and DTB, boots via EFI handoff.

On AArch64 with dracut initramfs, UUID-based root references work correctly
in both the BLS entry and fstab (unlike RISC-V no-initrd builds which require
explicit device paths).

---

### Stage 6 — Linux kernel (AArch64)

Built from mainline linux using a custom DTS derived from the upstream
`rk3566-tinker-board-3.dtsi`. AArch64 uses a dracut-generated initramfs,
so storage drivers can be modules — no `=y` constraint unlike RISC-V.

**Current hold status:** Ethernet (gmac1, RTL8211F) fails with DMA reset
errors under both kernel 6.19.8 and 7.0-rc4. All known DTS suspects have
been cleared (pinctrl gmac1m0 confirmed against schematic, RGMII delays,
clock configuration, voltage). Root cause unknown — question posted to
`#linux-rockchip` on Libera IRC, awaiting response.

The board boots cleanly to login with networking disabled. Ethernet TX+RX
works on the BSP vendor kernel — confirming the hardware is good. The DMA
reset failure appears to be a mainline kernel/DTS interaction specific to
this board.

---

### Stage 7 — Fedora 43 aarch64

Standard Fedora 43 aarch64 rootfs. `dnf update kernel` works correctly via
kernel-install + BLS — this is one of the advantages of the AArch64 + initramfs
path over the RISC-V no-initrd approach.

---

### Known-Good Baseline

BSP kernel (6.6.89) + BSP DTB → boots to login, ethernet working.
Mainline kernel (6.19.8, 7.0-rc4) + mainline DTS → boots to login, ethernet DMA reset failure.

---

### Open Items

- **Ethernet DMA reset** — root cause under investigation, `#linux-rockchip` Libera IRC
- **Upstream DTS contribution** — the ethernet node additions will be submitted
  upstream once the DMA issue is resolved and the configuration is confirmed correct
