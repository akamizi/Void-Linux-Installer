# Void Linux Installer

An opinionated, automated installer for Void Linux with full disk encryption,
BTRFS subvolumes, and direct UEFI boot (no GRUB).

---

> ## ⚠ UNTESTED — READ BEFORE USING
>
> **This script has not been tested end-to-end on real hardware yet.**
>
> It is the result of a careful review and rewrite of existing guides and
> scripts, but it has **not** gone through a full installation run. There may
> be bugs, incorrect assumptions, or compatibility issues with specific
> firmware or hardware configurations.
>
> **Do not run this script unless you:**
> - understand partitioning, LUKS2, BTRFS subvolumes, and UEFI boot
> - know how to recover a broken install from a live environment
> - have read and understood every section of both `void-installer` and
>   `void-installer-chroot` before executing them
> - have a way to verify each step independently (e.g. against the
>   [Void Linux documentation](https://docs.voidlinux.org))
>
> Contributions, test reports, and bug reports are very welcome.

---

> **Warning** — This script is intentionally destructive. It will wipe the
> target disk without further confirmation beyond the initial prompt.
> Read and understand the code before running it.

---

## What it does

| Layer | Choice | Rationale |
|-------|--------|-----------|
| Partition table | GPT, 2 partitions | EFI (1 GB FAT32) + encrypted root |
| Encryption | LUKS2 + AES-256-XTS + **argon2id** | argon2id is memory-hard; far stronger than the pbkdf2 used by LUKS1/GRUB |
| Filesystem | BTRFS | Subvolumes, transparent compression, snapshot support |
| Bootloader | **efistub** (no GRUB) | UEFI loads the kernel directly; smaller attack surface, faster boot |
| Init | runit (Void default) | — |

### Disk layout

```
/dev/nvme0n1
├── p1   1 GB   FAT32   /boot          EFI partition (kernel + initramfs, readable by UEFI)
└── p2   ~999 GB  LUKS2   /            Everything else, encrypted
                  └── BTRFS
                      ├── @            /
                      ├── @home        /home
                      ├── @swap        /swap          (swapfile, COW disabled)
                      ├── @snapshots   /.snapshots
                      ├── @var_log     /var/log       (excluded from snapshots)
                      └── @var_cache_xbps  /var/cache/xbps  (nested, excluded from snapshots)
```

`/tmp` is a `tmpfs` (RAM-backed, cleared on reboot, `noexec`/`nosuid`/`nodev`).

### Optional features

| Flag | Default | Description |
|------|---------|-------------|
| `ENABLE_UKI` | prompt | Bundle kernel + initramfs + cmdline into a single signed `.efi` (Unified Kernel Image) |
| `ENABLE_SECUREBOOT` | prompt | Generate custom PK/KEK/db keys, sign the UKI, prepare UEFI enrollment |
| `ENABLE_SNAPPER` | prompt | Configure automatic BTRFS snapshots for `/` and `/home` |
| `KEYMAP` | `it` | Console keyboard layout |

---

## Prerequisites

### Hardware

- x86\_64 system with **UEFI firmware** (no Legacy/CSM)
- Target disk: NVMe recommended (the BTRFS mount options are tuned for SSDs)
- At least 8 GB RAM recommended (argon2id uses 512 MB during LUKS unlock; 4 GB
  reserved for `/tmp`)

### Firmware settings (before booting the live ISO)

1. Enter your UEFI setup (usually **F2**, **Del**, or **F12** at power-on).
2. Set boot mode to **UEFI only** — disable Legacy/CSM if present.
3. **Secure Boot**: the Void Linux live ISO is not signed, so Secure Boot must be
   handled before booting:
   - If you plan to **enable Secure Boot** after install (`ENABLE_SECUREBOOT=true`):
     go to *Security → Secure Boot → Clear/Delete Keys* to enter **Setup Mode**
     now. This lets the installer enroll your custom keys automatically.
   - Otherwise: simply **disable Secure Boot** for the installation.
4. Set the USB drive as the first boot device.

---

## Step 0 — Create a bootable USB

Download the latest **x86\_64** Void Linux live image (base or xfce) from
[voidlinux.org/download](https://voidlinux.org/download/).
Verify the checksum, then write the image to a USB drive:

```sh
# Linux
dd if=void-live-x86_64-*.iso of=/dev/sdX bs=4M status=progress conv=fdatasync

# Windows / macOS
# Use Balena Etcher or Rufus (ISO mode, not DD mode on Rufus)
```

---

## Step 1 — Boot and connect to the internet

1. Boot from the USB drive.
2. Log in as `root` (no password on the live ISO).
3. Connect to the network:

```sh
# Wired ethernet — usually works automatically via dhcpcd
ip link   # confirm the interface is UP

# WiFi
wpa_passphrase "SSID" "password" > /etc/wpa_supplicant/wpa_supplicant.conf
wpa_supplicant -B -i wlan0 -c /etc/wpa_supplicant/wpa_supplicant.conf
dhcpcd wlan0
```

4. Confirm connectivity:

```sh
ping -c 3 voidlinux.org
```

---

## Step 2 — Install required tools and clone the repo

The script checks for missing dependencies at startup and will tell you exactly
what to install. On a minimal live ISO you typically need:

```sh
xbps-install -Syu xbps            # update xbps itself first
xbps-install -Sy git gptfdisk     # git + sgdisk (always needed)
```

Then clone this repository:

```sh
git clone https://github.com/akamizi/Void-Linux-Installer
cd Void-Linux-Installer
```

---

## Step 3 — Read the script

```sh
less void-installer
less void-installer-chroot
```

Understand what the scripts do before running them. The most important
tunables are at the top of `void-installer`:

| Variable | Default | Notes |
|----------|---------|-------|
| `DISK_NAME` | prompted | e.g. `/dev/nvme0n1` — verify with `lsblk` |
| `HOSTNAME` | prompted | |
| `USERNAME` | prompted | |
| `CPU_VENDOR` | auto-detected | `intel` or `amd` (for microcode) |
| `LOCALE` | `it_IT.UTF-8` | |
| `TIMEZONE` | `Europe/Rome` | |
| `KEYMAP` | `it` | |
| `SWAP_SIZE` | `8` | swapfile size in GB |
| `ENABLE_UKI` | prompted | |
| `ENABLE_SECUREBOOT` | prompted | requires `ENABLE_UKI=true` |
| `ENABLE_SNAPPER` | prompted | |
| `EXTRA_PACKAGES` | _(unset)_ | array of additional packages to install |

All variables can be set as environment variables to skip the interactive
prompts, e.g.:

```sh
DISK_NAME=/dev/nvme0n1 HOSTNAME=myvoid USERNAME=alice ENABLE_UKI=true ./void-installer
```

---

## Step 4 — Run the installer

```sh
./void-installer
```

The script will:

1. Check all required tools are present on the live system.
2. Ask for the disk encryption passphrase (typed twice, not echoed).
3. Show a summary and ask you to type `CONFERMO` before touching the disk.
4. Partition, format, encrypt, create BTRFS subvolumes, install the base system,
   and configure everything inside a chroot.
5. Prompt you to set passwords for the new user and root.
6. Save a **LUKS2 header backup** to `/tmp/luks-header-<timestamp>.bak`.

The entire process takes 5–15 minutes depending on your internet speed.

### ⚠ Save the LUKS2 header backup before rebooting

The header backup is the only way to recover your data if the LUKS2 header
gets corrupted (bad sectors, interrupted write). The installer saves it to
`/tmp` on the live system (volatile — lost on reboot).

```sh
# Copy to a USB drive before rebooting
cp /tmp/luks-header-*.bak /run/media/<usb>/

# To restore a corrupted header later:
# cryptsetup luksHeaderRestore /dev/nvme0n1p2 --header-backup-file luks-header.bak
```

---

## Step 5 — Reboot

```sh
umount -R /mnt
cryptsetup close cryptroot
reboot
```

Remove the USB drive when prompted (or just after the screen goes blank).

---

## After installation

### First boot

- The LUKS2 passphrase is asked once by the initramfs (dracut).
- If Secure Boot enrollment was not completed automatically, follow the
  instructions printed at the end of the install.

### Useful commands

```sh
efibootmgr -v                        # inspect UEFI boot entries
xbps-install -Su                     # system update (EFI automatically updated by hook)
snapper -c root list                 # list snapshots (if Snapper enabled)
snapper -c root create --description "before experiment"   # manual snapshot
cat /etc/efistub.conf                # boot configuration baked by the installer
cat /etc/kernel/cmdline              # embedded kernel cmdline (UKI mode only)
```

### Updating the embedded kernel cmdline (UKI mode)

Because the cmdline is compiled into the UKI binary, changing it requires
a rebuild:

```sh
# 1. Edit the cmdline
vim /etc/kernel/cmdline

# 2. Rebuild the UKI and update the UEFI entry
kver=$(ls /boot/vmlinuz-* | sort -V | tail -1 | sed 's|/boot/vmlinuz-||')
/etc/kernel.d/post-install/10-efistub "$kver"
```

### Secure Boot — manual key enrollment

If automatic enrollment failed during install, the `.auth` files are already
on the EFI partition at `/boot/EFI/secureboot/`. Follow the instructions
printed at the end of the install, or re-read them from the chroot script
comments.

---

## Security notes

- **LUKS2 + argon2id**: 512 MB RAM, 4 threads, 3 s calibration. Resisting
  GPU-based brute force is orders of magnitude harder than with pbkdf2.
- **`/tmp`**: `tmpfs` with `noexec`/`nosuid`/`nodev` — executables dropped
  here cannot be run directly.
- **`/boot/EFI`**: mounted `noexec`/`nosuid`, `umask=0077` — only root can
  read kernel and initramfs files.
- **`/var/log`**: separate subvolume — restoring a snapshot of `/` does not
  roll back logs (forensic integrity).
- **sysctl hardening**: `dmesg_restrict`, `kptr_restrict`, `randomize_va_space`,
  `protected_hardlinks/symlinks`, SYN cookies, RP filter, ICMP redirect
  suppression.
- **SSH**: pre-configured to reject root login and password auth (takes effect
  if/when `openssh` is installed).

---

## Acknowledgements

This installer was built on the shoulders of:

- **[gbrlsnchs](https://gist.github.com/gbrlsnchs/9c9dc55cd0beb26e141ee3ea59f26e21)** —
  the original gist that established the LUKS + BTRFS subvolume layout and
  dracut configuration patterns this script evolved from.

- **[Void Linux documentation](https://docs.voidlinux.org/installation/guides/fde.html)** —
  the official FDE installation guide covering LUKS, keyfile setup, dracut
  integration, and crypttab conventions.

- **[Jesse Garcia](https://jesseg.co/articles/installing-void-linux/)** —
  for the NVMe-specific BTRFS mount options, the UKI/efistub approach, and
  the gummiboot-based dracut workflow.

- **[NicholasBHubbard](https://github.com/NicholasBHubbard/Void-Linux-Installer)** —
  the original shell script that served as the starting point and structural
  template for this version. This repository is a heavily revised fork of that work.
