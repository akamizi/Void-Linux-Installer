# Void Linux Light Installer

A simpler alternative to `void-installer` for cases where full disk
encryption and Secure Boot are not required.

---

> **Warning** — This script is intentionally destructive. It will wipe the
> target disk without further confirmation beyond the initial prompt.
> Read and understand the code before running it.

---

## What it does

| Layer | Choice |
|-------|--------|
| Partition table | GPT, 2 partitions: EFI (1 GB FAT32) + root |
| Encryption | **None** |
| Filesystem | **BTRFS** (subvolumes) or **ext4** (flat) — your choice |
| Bootloader | **efistub** (no GRUB, no UKI, no Secure Boot) |
| Desktop | **Fluxbox** + Xorg (minimal WM, no display manager) |
| Init | runit (Void default) |

### Disk layout — BTRFS

```
/dev/nvme0n1
├── p1   1 GB   FAT32   /boot
└── p2   ~∞     BTRFS   /
                └── subvolumes
                    ├── @            /
                    ├── @home        /home
                    ├── @swap        /swap          (swapfile, COW disabled)
                    ├── @snapshots   /.snapshots
                    ├── @var_log     /var/log
                    ├── @var_cache_xbps  /var/cache/xbps
                    └── @srv         /srv
```

Same BTRFS mount options as the main installer: `zstd:1` compression,
`discard=async`, `space_cache=v2`, `noatime`.

### Disk layout — ext4

```
/dev/nvme0n1
├── p1   1 GB   FAT32   /boot
└── p2   ~∞     ext4    /           (everything here, including /home)
                         /swap/swapfile
```

`/tmp` is a `tmpfs` in both cases.

---

## Prerequisites

Same as `void-installer`: x86\_64 system with UEFI firmware, no
Legacy/CSM. No specific firmware preparation required (no Secure Boot
setup needed).

On the live ISO:

```sh
xbps-install -Syu xbps
xbps-install -Sy git gptfdisk dosfstools btrfs-progs   # for BTRFS
xbps-install -Sy git gptfdisk dosfstools e2fsprogs      # for ext4
```

---

## Usage

```sh
git clone https://github.com/akamizi/Void-Linux-Installer
cd Void-Linux-Installer
./void-installer-light
```

All settings can be pre-set via environment variables to skip the
interactive prompts:

```sh
DISK_NAME=/dev/sda FS_TYPE=ext4 HOSTNAME=mybox USERNAME=alice ./void-installer-light
```

| Variable | Default | Notes |
|----------|---------|-------|
| `DISK_NAME` | prompted | e.g. `/dev/nvme0n1` |
| `HOSTNAME` | prompted | |
| `USERNAME` | prompted | |
| `CPU_VENDOR` | auto-detected | `intel` or `amd` (for microcode) |
| `FS_TYPE` | prompted | `btrfs` or `ext4` |
| `LOCALE` | `it_IT.UTF-8` | |
| `TIMEZONE` | `Europe/Rome` | |
| `KEYMAP` | `it` | |
| `SWAP_SIZE` | `8` | swapfile size in GB |

---

## What gets installed

**Base system:** `base-system`, `efibootmgr`, `btrfs-progs` (BTRFS only).

**Desktop:** Fluxbox, Xorg, `xinit`, `xterm`, `feh` (wallpaper), `dmenu`
(launcher), `dejavu-fonts-ttf`. No display manager.

**Networking:** `dhcpcd`. For WiFi, install `wpa_supplicant` after the
first boot.

**Hardening** (applied automatically):

- `umask 027` — new files not readable by other users
- SSH pre-hardened: `PermitRootLogin no`, `PasswordAuthentication no`,
  `X11Forwarding no`, `AuthenticationMethods publickey` (takes effect
  if/when `openssh` is installed)
- sysctl: `dmesg_restrict`, `kptr_restrict`, ASLR max,
  `protected_hardlinks/symlinks`, SYN cookies, RP filter, ICMP redirect
  suppression

---

## First boot

Log in as your user on tty1. X starts automatically (via
`/etc/profile.d/50-startx.sh`). To disable auto-start:

```sh
sudo rm /etc/profile.d/50-startx.sh
```

Then start manually with `startx`.

**Fluxbox basics:**

- Right-click the desktop → application menu
- Middle-click → window list
- `dmenu` is available from the menu or via keyboard shortcut (configure
  in `~/.fluxbox/keys`)

**WiFi setup** (if needed):

```sh
sudo xbps-install -Sy wpa_supplicant
wpa_passphrase "SSID" "password" | sudo tee /etc/wpa_supplicant/wpa_supplicant.conf
sudo wpa_supplicant -B -i wlan0 -c /etc/wpa_supplicant/wpa_supplicant.conf
sudo dhcpcd wlan0
```

---

## Updating the kernel cmdline

Unlike UKI mode, the kernel cmdline is stored as plain text in
`/etc/efistub.conf` and can be changed at any time:

```sh
# 1. Edit KERNEL_CMDLINE in /etc/efistub.conf
sudo vim /etc/efistub.conf

# 2. Re-run the hook to update the EFI entry
kver=$(ls /boot/vmlinuz-* | sort -V | tail -1 | sed 's|/boot/vmlinuz-||')
sudo /etc/kernel.d/post-install/10-efistub "$kver"
```

---

## Differences from void-installer

| Feature | `void-installer` | `void-installer-light` |
|---------|-----------------|----------------------|
| Disk encryption | LUKS2 + argon2id | None |
| Filesystem | BTRFS only | BTRFS or ext4 |
| Secure Boot | Optional | Not available |
| UKI | Optional | Not available |
| Snapper | Optional | Not available |
| Desktop | Optional (Cinnamon) | Fluxbox (always) |
| Networking | dhcpcd or NetworkManager | dhcpcd |
