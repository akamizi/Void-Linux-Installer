# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A two-stage Bash installer for Void Linux with full disk encryption (LUKS2/argon2id), BTRFS subvolumes, and direct UEFI boot (efistub or UKI). No build system — the scripts run directly on a Void Linux live ISO.

## Running the Installer

```bash
# Interactive mode (prompts for all settings)
./void-installer

# Non-interactive via environment variables
DISK_NAME=/dev/nvme0n1 HOSTNAME=myvoid USERNAME=alice ENABLE_UKI=true ./void-installer
```

**Prerequisites on live ISO:**
```bash
xbps-install -Sy git gptfdisk
# Optional: btrfs-progs cryptsetup dosfstools efibootmgr gummiboot binutils
```

## Two-Stage Architecture

### Stage 1: `void-installer` (runs on live ISO)
1. Dependency validation (`check_deps()`)
2. Interactive configuration collection with validation
3. GPT disk partitioning: `p1` = 1 GB FAT32 EFI, `p2` = LUKS2-encrypted root
4. LUKS2 setup with AES-256-XTS + argon2id KDF
5. BTRFS with 7 subvolumes (`@`, `@home`, `@snapshots`, `@var_log`, `@var_cache`, `@var_tmp`, `@swap`)
6. Mount subvolumes with security options (`nodev`, `nosuid`, etc.)
7. Base system bootstrap via `xbps-install`
8. Transition into chroot, calling `void-installer-chroot`

Trap-based cleanup unmounts filesystems and closes LUKS on any error or interrupt.

### Stage 2: `void-installer-chroot` (runs inside new system)
Configured by environment variables exported from Stage 1:
- Locale, hostname, timezone, keymap
- BTRFS swapfile (CoW disabled)
- `/etc/fstab` generation (UUID-based)
- Dracut initramfs with crypt+btrfs modules, zstd compression
- **Optional UKI** (`ENABLE_UKI=true`): bundles kernel+initramfs+cmdline into single `.efi`
- **Optional Secure Boot** (`ENABLE_SECUREBOOT=true`): RSA4096 key generation (PK/KEK/db), UKI signing, UEFI enrollment attempt
- **Optional Snapper** (`ENABLE_SNAPPER=true`): automated BTRFS snapshots on `/` and `/home`
- **Optional Desktop** (`ENABLE_DESKTOP=true`): calls `/void-installer-desktop` (see below), replaces dhcpcd with NetworkManager
- `/etc/kernel.d/post-install/10-efistub` hook: auto-rebuilds EFI on kernel updates
- Intel/AMD microcode detection and installation
- User creation, sudo/wheel membership
- sysctl hardening (ASLR, dmesg_restrict, kptr_restrict)
- SSH pre-hardening (no root login, no password auth)

## `void-installer-desktop`

Separate script called from `void-installer-chroot` when `ENABLE_DESKTOP=true`. Installs Cinnamon desktop environment as a faithful replica of the current system:
- Packages: `cinnamon-all`, `nemo` + extensions, `xfce4-terminal`, `xorg-server`, `greetd`, `elogind`, `seatd`, `pipewire`, `wireplumber`, `alsa-pipewire`, `NetworkManager` + applet, `polkit-gnome`, `gnome-keyring`, `power-profiles-daemon`, `touchegg`, base fonts
- Configures `/etc/greetd/config.toml` (agreety + cinnamon-session)
- Sets up ALSA→PipeWire symlinks in `/etc/alsa/conf.d/`
- Enables services: `dbus`, `elogind`, `seatd`, `NetworkManager`, `greetd`, `power-profiles-daemon`, `touchegg`, `pipewire` (system-level)
- Removes `dhcpcd` symlink (replaced by NetworkManager)

PipeWire runs as a system service (simpler with runit, suitable for single-user desktops). `wireplumber` starts per-user via Cinnamon's XDG autostart.

## Key Generated Files

| File | Purpose |
|------|---------|
| `/etc/dracut.conf.d/10-base.conf` | Initramfs config |
| `/etc/dracut.conf.d/20-uki.conf` | UKI-specific config (if enabled) |
| `/etc/kernel/cmdline` | Kernel cmdline embedded in UKI |
| `/etc/efistub.conf` | Read by kernel update hook |
| `/etc/kernel.d/post-install/10-efistub` | Post-kernel-update EFI hook |
| `/etc/secureboot/{PK,KEK,db}.{key,crt}` | Secure Boot keys (if enabled) |
| `/boot/EFI/void/void-linux.efi` | UKI binary (if UKI mode) |

## Coding Conventions

- Comments are written in **Italian** — maintain this convention
- Every non-obvious design choice includes inline rationale (security tradeoffs, format choices)
- Use `set -euo pipefail` error handling throughout
- Prefer UUIDs over device names in generated configs
- Confirmation prompts before destructive operations
- The `ENABLE_*` boolean environment variables control optional features; they default to `false` when unset

## Testing

No automated test suite. Testing is manual, on real hardware. The project has been validated on a Lenovo IdeaPad 710S. Any change to partitioning, encryption, or EFI logic should be tested end-to-end on a VM or spare hardware before merging.
