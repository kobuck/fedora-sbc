# fedora-sbc

Fedora on single-board computers — annotated boot chains, kernel configs,
and build notes from working systems.

These are reference documents for people building Fedora on SBC hardware,
not tutorials or maintained software. The goal is that someone working on
the same or similar hardware finds useful content rather than vendor
documentation and silence.

## Boards

| Board | SoC | Arch | Status |
|-------|-----|------|--------|
| [Milk-V Mars](milkv-mars/) | StarFive JH7110 | riscv64 | ✅ Working — Fedora 43 |
| [Orange Pi RV](orangepi-rv/) | StarFive JH7110 | riscv64 | ✅ Working — Fedora 43 |

## What's Here

Each board directory contains:
- **boot-chain.md** — annotated description of every boot stage from power-on
  to running OS, explaining what each stage does and why it matters
- Additional notes on kernel config, DTS, and lessons learned as they accumulate

## Common JH7110 Gotchas

Both boards share the same boot chain structure (Boot ROM → SPL → OpenSBI →
U-Boot → systemd-boot → kernel → Fedora). Key things that trip people up:

- **Partition type GUIDs** — the Boot ROM locates bootloader partitions by
  type GUID, not partition number. Wrong GUIDs = silent failure, board appears dead
- **No initrd on RISC-V** — storage drivers must be built-in (`=y`), not modules
- **EFI variable store** — U-Boot uses a file on the ESP (`ubootefi.var`);
  `CONFIG_EFI_RT_VOLATILE_STORE=y` is required for writable efivarfs
- **SPI NOR reflash** — use U-Boot `sf update`, never `dd` to `/dev/mtdX`

## Build Environment

Builds use mainline sources on a RISC-V builder VM (x86_64 host, riscv64
cross-compiler):
- U-Boot v2026.01
- OpenSBI v1.8.1
- Linux v6.19
- Fedora 43 riscv64 ([riscv-kojipkgs.fedoraproject.org](https://riscv-kojipkgs.fedoraproject.org/repos/f43-build/latest/riscv64/))
