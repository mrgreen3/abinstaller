# abinstaller

ArchBang system installer - a comprehensive bash-based installation script for ArchBang Linux.

## Overview

`abinstaller` (abinstall) is an interactive installer script that automates the installation and configuration of ArchBang on your system. It handles:

- Disk partitioning (with support for LVM and LUKS encryption)
- Filesystem creation and mounting
- ArchBang system installation
- System configuration (hostname, timezone, locale, keyboard)
- Bootloader setup (GRUB2 with UEFI/BIOS support)
- User account creation

## Features

✅ **Disk Management**
- Partition creation with multiple tools (fdisk, gdisk, parted, gparted, cfdisk, cgdisk)
- LVM (Logical Volume Manager) support
- LUKS full-disk encryption with LVM+LUKS option
- Automatic TRIM detection for SSDs

✅ **System Configuration**
- Automatic locale and timezone detection
- Keyboard layout selection (system and desktop)
- Custom hostname configuration
- Mirror selection by geographic location

✅ **Bootloader**
- GRUB2 installation for both UEFI and BIOS systems
- Automatic LUKS/LVM kernel parameter generation
- Microcode updates for Intel/AMD processors

✅ **User Management**
- Root password setup
- New user creation with sudo access
- Live user cleanup
- Sudo hardened post-install: live ISO uses passwordless sudo (`NOPASSWD`) for convenience; installer reverts this so the installed system requires a password for all sudo operations

## Usage

```bash
# Basic interactive mode
sudo ./abinstall

# With logging
sudo ./abinstall --log

# With pre-configured values
sudo ./abinstall hostname=myarch locale=en_US.UTF-8 keymap=us

# Interactive installation (skip prompts)
sudo ./abinstall --install
```

## Requirements

- Running from ArchBang live environment
- Root access
- Arch Linux derivatives (tested on ArchBang)

## Files

- **abinstall** - Main installation script

## License

GPL-3.0-or-later - See LICENSE file for details

## Credits

- Original author: helmuthdu (helmuthdu@gmail.com)
- Contributors: flexiondotorg
- Current maintainer: Mr Green (mrgreen@archbang.org)

## References

- [ArchBang Project](https://www.archbang.org)
