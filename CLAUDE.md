# CLAUDE.md — ArchBang Installer Context

This file is context for an AI agent working on this codebase. Read it before making any changes.

---

## Project Purpose and Philosophy

`abinstall` is a terminal-based system installer for ArchBang Linux — an Arch Linux derivative using the Labwc Wayland compositor. ArchBang's goal is to give new Arch users a gentle introduction: a minimal, working environment without overwhelming complexity.

The installer originated from the FIFO/helmuthdu bash installer and has evolved organically over 20+ years, with recent AI-assisted modifications by Mr Green (mrgreen@archbang.org). **Refactoring must be incremental and cautious.** Working functionality takes priority over structural elegance. Do not add complexity. Do less rather than more.

The installer is designed to run exclusively from the ArchBang live ISO environment.

---

## Repository Contents

- `abinstall` — The main installer script (~1300 lines of bash)
- `mvuser` — User rename script, runs inside the chroot at `/root/mvuser` (see below)
- `README.md` — User-facing overview

There are no config files, no tests, no build system. `mvuser` ships on the live ISO at `/root/` and is copied to the installed system as part of the airootfs copy in step 2.

---

## What the Installer Does (Step by Step)

The installer runs as root from the live environment. It runs each step in a fixed linear sequence via `run_step`. Steps already marked complete in `checklist[]` are skipped automatically within a session. If a step fails, the user is offered retry or skip-and-continue before the installer proceeds.

### Startup (automatic)
1. Parses CLI args: `-l/--log`, `-i/--install`, `hostname=`, `locale=`, `keymap=`
2. Sets global variables (colors via `tput`, paths, flags)
3. Runs: `check_root`, `check_boot_system` (UEFI/BIOS detection), `detect_cpu` (Intel/AMD), `check_trim` (SSD detection)

### Menu Items
| # | Label | Functions called |
|---|-------|-----------------|
| 1 | Partition Scheme | `umount_partitions`, `create_partition_scheme`, `format_partitions` |
| 2 | Install ArchBang | `install_archbang`, `configure_fstab`, `configure_mkinitcpio` |
| 3 | Hostname | `configure_hostname` |
| 4 | Location | `configure_timezone`, `configure_hardwareclock`, `configure_mirrors` |
| 5 | Locale | `configure_locale` |
| 6 | Keyboard Layout | `configure_vconsole`, `select_xkeymap` |
| 7 | Bootloader | `configure_bootloader` |
| 8 | Root and User Setup | `root_password`, `setup_user` |
| d | Done | `finish` (optionally reboots) |

### Key Operations Per Step
- **Step 1**: Offers Default/LVM/LVM+LUKS/Maintain Current layouts. Opens a partition tool (gparted, cfdisk, cgdisk, fdisk, gdisk, parted), then formats and mounts partitions.
- **Step 2**: Copies live airootfs to `/mnt` via tar+pv, syncs cowspace overlays, generates machine-id, writes mkinitcpio preset, strips live-only artifacts (autologin, pacman-init, skel, installer menu entries), regenerates initramfs.
- **Step 4**: Timezone selection, hardware clock (always UTC), mirror selection by country.
- **Step 7**: GRUB2 install — UEFI (`grub-install --target=x86_64-efi`) or BIOS (`grub-install --target=i386-pc`). Builds kernel cmdline with LUKS/LVM params. Runs `grub-mkconfig`.
- **Step 8**: Sets root password via `passwd` in chroot. Then runs `mvuser` in chroot if present, otherwise falls back to inline `useradd`. See `mvuser` behaviour below.

### How `mvuser` Works

`mvuser` is a `/bin/sh` script that runs **inside the chroot** after install. It does not create a new user — it **renames the existing `ablive` live user** to whatever the installer user provides:

1. Checks that `/home/ablive` exists; if not, exits (rename already done or user missing)
2. Prompts for a new username, validates it (`^[a-z_][a-z0-9_-]*[$]?$`)
3. Sets the password on the `ablive` account (`passwd ablive`) — this becomes the new user's password
4. Runs `sed -i "s/$usr1/$usr2/g"` on every file inside `/home/ablive/` — replaces all occurrences of the old username in config files
5. Updates `/etc/group`, `/etc/gshadow`, `/etc/passwd`, `/etc/shadow` directly via sed
6. Renames `/home/ablive` → `/home/newusername`
7. `chown -R newusername:users /home/newusername`

**Important implication**: Any installer step that writes config to `$MNT/home/${LIVE_USER}/...` (e.g. `select_xkeymap` writing to `$MNT/home/ablive/.config/labwc/environment`) is correct by design — `mvuser` will rename the directory and update config file contents to the new username.

`mvuser` is a one-shot operation. If run a second time it exits immediately.

---

## Known Gaps and Broken Areas

### Fragilities

- No enforcement of menu step ordering. Running step 7 (bootloader) before step 2 (install) will fail badly.
- `copy_progress` requires `pv` to be installed in the live environment. No fallback if missing.
- `format_partition` runs `fsck` immediately after `mkfs` — unnecessary and occasionally problematic on freshly formatted partitions.
- The broad ERR trap (`umount_partitions` on any error) can cause collateral damage if a non-fatal command fails.
- Hardcoded defaults: `ROOT_MOUNTPOINT="/dev/sda1"`, `BOOT_MOUNTPOINT="/dev/sda"`, `LUKS_DISK="sda2"` — wrong on most modern hardware (nvme, virtio, etc.).
- `check_archiso` checks for `/run/archiso/airootfs` — blocks any testing outside the live environment.

---

## Assumptions About the Target System

- Running from ArchBang live ISO (x86_64)
- Live user account is named `ablive` (hardcoded as `LIVE_USER`)
- Labwc config exists at `~/.config/labwc/` in the live user's home
- `pv`, `gparted`, `cfdisk`, `cgdisk`, `fdisk`, `gdisk`, `parted` are available in the live environment
- Target disk mounted at `/mnt`
- Kernel package is `linux` (mkinitcpio preset hardcoded to `linux.preset`)
- GRUB is the bootloader (no systemd-boot support)
- CPU is Intel or AMD (for microcode detection)
- `mvuser` ships on the live ISO at `/root/mvuser` and is copied into the installed system via the airootfs copy in step 2

---

## Codebase Conventions

- **Fold markers**: Functions use `#{{{` and `#}}}` (vim fold markers). Preserve these.
- **`arch_chroot`**: Wrapper around `arch-chroot $MNT /bin/bash -c "${1}"`. Used for all operations inside the installed system.
- **`read_input_text`**: Yes/no prompts, normalises to lowercase. Returns in `$OPTION`.
- **`read_input_options`**: Multi-option input (supports ranges like `1-3`). Returns array in `$OPTIONS`.
- **`checklist[]`**: Array tracking completed steps. Set to 1 after each menu item runs. Display only — not enforced.
- **Color variables**: Set via `tput` at startup (Bold, Red, Green, etc. and their Bold variants BRed, BGreen...).
- **`MNT="/mnt"`**: All target filesystem operations use this prefix.
- **Global state variables**: `UEFI`, `LVM`, `LUKS`, `TRIM`, `cpu_id` — set at startup and used throughout. `LUKS_PART`, `LUKS_MAPPER`, `ROOT_PART` set during partitioning and used in bootloader config.

---

## Open Questions

1. **Should the mirror step happen before or after install?** Currently it's in the same menu item as timezone (step 4) but requires step 2 to have populated `/mnt/etc/pacman.d/mirrorlist`.
