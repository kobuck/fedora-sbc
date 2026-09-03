# GPT Image Creation — Building These Boot Chains

This describes the general process used to build every image in this
repository: how the GPT partition layout is designed, why raw-sector
bootloaders and GPT partition tables coexist without conflict, and the
build sequencing that avoids several non-obvious failure modes. Board pages
link back here rather than repeating this material per-board; board pages
describe only what is specific to that SoC.

This document describes the current template's target implementation. As
the template evolves, previous Home Lab builds may not conform to the
current version — the intention is that the template is applied in the
normal course of SBC build image maintenance (e.g. the next major Fedora
release build for a given board), not retrofitted onto boards that are
already working.

## Why GPT at all, when some of these Boot ROMs read raw sectors?

Every board in this repository ends up handing off to U-Boot's EFI
subsystem, which locates the ESP (EFI System Partition) by scanning the GPT
for its standard type GUID. **GPT is required for the boot chain to work at
all, regardless of whether the earlier Boot ROM/SPL stage itself reads GPT
or a raw sector offset.** MBR is not a viable alternative for any board
here, even the ones (RK3399, RK3566, H618) whose Boot ROM ignores partition
tables entirely and reads a fixed raw sector.

This means every image has two independent addressing systems that must
both be satisfied simultaneously:
- The **Boot ROM/SPL stage**, which on most boards here reads a raw sector
  offset with no awareness of any partition table at all
- **Everything from U-Boot's EFI layer onward**, which requires a real,
  valid GPT

The partition layout below is designed so both can be true at once — by
placing GPT partition *entries* over the raw regions purely as protective
fencing (metadata only, no filesystem), so disk tools can see and avoid
those regions without any tool needing to understand the raw sector
convention itself.

## The Four Storage Entities

Every image in this repository represents the same conceptual storage
entities:

- **SPL / initial bootloader stage** — DDR init + first-stage bootloader,
  loaded directly by the Boot ROM
- **U-Boot proper** — the full bootloader with EFI support
- **ESP** — standard FAT32 EFI System Partition; kernels, DTBs, initrds,
  BLS entries, systemd-boot binary
- **RootFS** — the actual Fedora userspace

Of these, only the ESP and RootFS are formal GPT partitions in every
image — SPL and U-Boot are raw sector writes that a SoC's Boot ROM expects
at fixed or discoverable locations, not filesystem-based partitions in the
traditional sense. To keep this raw bootloader space from being invisible
dead space that any partitioning tool could silently overwrite, we allocate
a fixed 16MB pre-ESP envelope, reserved for whatever SoC/SBC-specific
firmware requirements a given board has.

## Template Partition Definitions

For boards using the pre-ESP firmware envelope as-is (see Template in
Practice below), the standard sector layout is:

| Partition | Start LBA | Size | End LBA | Role |
|---|---|---|---|---|
| p1 | 64 | ~4 MiB | 8,191 | SPL — pre-ESP envelope |
| p2 | 8,192 | 4 MiB | 16,383 | Spare — pre-ESP envelope |
| p3 | 16,384 | 4 MiB | 24,575 | U-Boot — pre-ESP envelope |
| p4 | 24,576 | 4 MiB | 32,767 | Spare — pre-ESP envelope |
| p5 | 32,768 | 2 GiB | 4,227,071 | EFI System Partition (ESP) |
| p6 | 4,227,072 | storage device specific | board-specific | RootFS |

p1–p4 together form the 16MB pre-ESP firmware envelope. The ESP always
starts at sector 32,768 and is always 2 GiB. RootFS is sized per board,
computed from that board's real measured storage capacity rather than a
fixed constant, so the same layout can serve both a build/validation SD
card and a smaller onboard eMMC or NVMe target identically.

**These definitions are subject to modification only as required to meet
specifically defined SoC/SBC board requirements.**

## Template in Practice

- **Rockchip RK3399** — uses the template as-is; no modification required.
- **Allwinner H618** — Boot ROM requires SPL at sector 16, forcing the GPT
  partition table to relocate into p2.
- **StarFive JH7110 (RISC-V)** — Boot ROM locates SPL and U-Boot by GPT
  partition type GUID rather than fixed sector offset, so p1–p4 carry
  vendor-specific GUIDs instead of raw positional requirements.
- **Raspberry Pi BCM2837** — does not use the pre-ESP envelope at all;
  implements a Hybrid MBR pointing to the ESP as its working boot
  partition.

Per-board implementation detail — exact sectors, partition GUIDs, and
build commands — is covered in each board's own implementation page, not
here.

## Build sequencing — the order that avoids silent corruption

1. **Zero any known bootloader-environment sector range first**, if
   rebuilding a device that previously had a valid U-Boot environment
   saved (identifiable by a valid CRC on the env block at its known
   offset). A stale environment from a previous build can persist across a
   full reflash and reintroduce old, unwanted settings.
2. **Partition the device** (`sgdisk`), placing fence entries over any raw
   bootloader regions.
3. **Format ESP and RootFS.**
4. **Assemble the rootfs into the real partitioned target** — mount ESP and
   RootFS into a real target tree, copy the built rootfs content in, write
   `/etc/fstab` with the target's real UUIDs.
5. **Run `kernel-install` (which invokes dracut) against the real, mounted
   target** — not against a flat, unpartitioned rootfs directory. This
   ordering matters more than it might appear: `kernel-install`'s BLS-entry
   logic decides where entries belong by checking the *live mount table*
   for a real mounted ESP at `/boot/efi`, not by reading `/etc/fstab` (which
   at this point may not even reference a live device yet in some build
   sequences). Running it too early, against a flat rootfs with no ESP
   actually mounted, produces a confusing failure rather than a working
   result.
6. **Write the bootloader raw-sector blobs (idbloader/SPL, U-Boot proper)
   as the absolute last step**, after every partition and filesystem
   operation is complete. Any partition tool run after this point risks
   silently destroying the blob — this is the single most consistent
   footgun across every board in this repository, regardless of SoC.
7. **Verify** — a full `gdisk -l`/`sgdisk -p` listing showing every
   partition with correct type GUIDs is the primary success criterion: the
   entire disk's contents should be self-documenting from that one command
   alone, with no need for external reference to know what occupies each
   region.

## Cross-arch chroot build gotchas (building on an x86_64 builder VM)

Every non-x86_64 image here is built on an x86_64 builder VM under
qemu-user-static emulation, not natively on the target hardware. This
introduces failure modes that don't occur when building or updating a
system natively:

- **`/etc/kernel/devicetree` must exist before the first `kernel-install`
  run**, containing the DTB's relative path
  (e.g. `rockchip/rk3399-rock-pi-4b-plus.dtb`). `kernel-install` does not
  auto-detect a board's DTB from any other source; without this file the
  resulting BLS entry silently has no `devicetree` line at all, and the
  kernel will not boot correctly despite `kernel-install` reporting success.
- **`/etc/kernel/cmdline` must exist before the first `kernel-install` run
  inside the chroot**, containing the target's correct
  `root=UUID=... console=... systemd.machine_id=...` values. A bare
  `chroot` does not create a new kernel or PID namespace — even with
  `/proc`, `/sys`, `/dev` correctly bind-mounted into the target,
  `kernel-install`'s BLS-writing logic falls back to reading the **host's**
  live `/proc/cmdline` whenever this file doesn't exist, silently baking
  the *builder VM's own* root UUID and machine-id into the BLS entry. The
  resulting entry looks entirely plausible (valid UUID and machine-id
  format) and this will only be caught by explicitly cross-checking against
  the target's actual `blkid` output — worth doing as a standard
  verification step on every build, not just when something looks wrong.
- **CA trust bundle may not be generated automatically** —
  `update-crypto-policies --set DEFAULT` (a standard rootfs setup step)
  does not itself regenerate `/etc/pki/ca-trust/extracted/pem/
  tls-ca-bundle.pem`. If this file is missing, all HTTPS-based `dnf`
  operations fail on first live boot with a TLS trust-anchor error. Run
  `update-ca-trust extract` explicitly as its own step.
- **Package post-install scriptlets that depend on a live D-Bus/systemd
  context may not complete correctly under emulation** — one confirmed
  example: `firewalld`'s PolicyKit action symlink
  (`/usr/share/polkit-1/actions/org.fedoraproject.FirewallD1.policy`, which
  polkit resolves to determine which policy variant is active) was absent
  from disk on a freshly-built rootfs despite the RPM database listing it
  as package-owned. This broke all `firewall-cmd` operations until
  `dnf reinstall firewalld` was run against the live, booted target (not
  the chroot) — the same scriptlet, run against a real running system
  rather than under qemu emulation, completed correctly. Worth a general
  verification pass (spot-check a few packages with D-Bus-dependent
  post-install steps) rather than assuming every scriptlet ran cleanly
  just because `dnf` reported no errors during the chroot build.

## Two-Target Builds (SD + onboard eMMC on the same board)

Some boards here (Rock Pi 4 Plus) have both a removable SD slot and
non-removable onboard eMMC. Building each independently risks introducing
drift between them and repeating diagnosis work on whichever target has
less error margin. The preferred approach instead:

1. **Build once, validate fully on the larger/more accessible target
   (typically SD)** — this is where iteration and debugging happen.
2. **Size RootFS as a fixed value** (not "remainder") from the start, sized
   to fit the *smaller* target's actual usable capacity (measured directly
   via `/sys/class/block/<dev>/size` or `blockdev --getsz` — always
   confirm actual capacity, which is consistently smaller than the nominal/
   marketing figure printed on the device) with reasonable margin. This
   way the identical partition table works on both targets without
   modification.
3. **Clone the validated image onto the second target**, executed live from
   the already-booted first target treating the second (often
   non-removable, no external programmer available) target as an inert
   block device.
4. **Re-randomize storage-level identifiers only** on the cloned copy — GPT
   disk GUID, all partition GUIDs, filesystem UUIDs (ext4 UUID, vfat volume
   serial). This step is mandatory, not optional: both devices remain
   simultaneously visible as block devices to whichever OS happens to be
   running, regardless of which one actually booted, so UUID-based
   resolution (`blkid`, `/etc/fstab`, BLS `options` lines) must be able to
   tell them apart.
5. **Leave machine-level identity shared** on the cloned copy — machine-id,
   SSH host keys, hostname. Only one of the two devices is ever the actual
   running system at any given moment (the non-booted device is simply
   inert storage), so there is no scenario requiring two distinct machine
   identities to coexist, and sharing them is both correct and simpler.
6. **Update every file that references the old storage UUIDs on the cloned
   copy** — `/etc/fstab`, every BLS entry's `options root=UUID=...` line,
   and `/etc/kernel/cmdline` (easy to miss, but required so that any
   *future* `kernel-install` run against the clone picks up the correct
   identity rather than reverting to the original target's values).

## Boot-device selection with two bootable devices — a real gotcha

Once a board has two independently bootable storage devices, U-Boot's
device-selection logic can surprise you. On U-Boot builds using
`bootcmd=bootflow scan -lb` (the modern bootflow/bootmeth mechanism), device
selection is governed by the **bootmeth priority list** (`bootmeths`), not
a strict walk of `boot_targets` in order. If `efi_mgr` (EFI-Boot-Manager-
style discovery) is present in `bootmeths`, it will select the first device
it finds with a valid bootable ESP — which may not be the device listed
first in `boot_targets`, if both devices happen to have valid ESPs
simultaneously (exactly the situation a two-target build like the one
above creates).

**Fix, applied once both devices are built and bootable, persisted via
`saveenv`:**
```
setenv bootmeths "extlinux script efi pxe"
saveenv
```
Removing `efi_mgr` forces U-Boot to respect `boot_targets` device order
using the remaining bootmeths. This is a one-time environment change that
survives reboots — not a per-boot workaround.

---

Board-specific pages: each links back here for the shared process and
documents only what differs for that SoC. See the per-board `boot-chain.md`
pages for full worked GUID tables, sector maps, and build commands.

Questions and corrections welcome — issues for specific problems,
discussions for questions and experience sharing.
