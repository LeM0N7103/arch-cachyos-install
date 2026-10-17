# Arch Linux Install via the CachyOS ISO

A manual Arch Linux install done from the **CachyOS live ISO**, using:

- **Btrfs** with subvolumes (`@`, `@/home`, `@/var`, plus No-CoW subvolumes for `AI` and `Games`)
- **linux-cachyos** kernel and the **CachyOS repositories** on top of plain Arch
- **Unified Kernel Image (UKI)** built by `mkinitcpio`, booted straight from UEFI (no GRUB / systemd-boot)
- **Secure Boot** with `sbctl`
- **systemd-networkd + systemd-resolved** for networking
- Selected **CachyOS-Settings** tweaks copied in by hand

> ⚠️ **This guide wipes the target disk.** Double-check every device name with `lsblk` before running anything.

---

## My setup (change these for yours)

| Item | Value used in this guide |
| --- | --- |
| Disk | `/dev/nvme0n1` |
| EFI System Partition (ESP) | `/dev/nvme0n1p1` (512 MiB, FAT32) |
| Root partition | `/dev/nvme0n1p2` (rest of disk, Btrfs) |
| Username | `lemon` |
| Hostname | `miraigadget` |
| Timezone | `Indian/Maldives` |
| Locales | `en_US.UTF-8`, `ja_JP.UTF-8` |
| Keymap | `us` |
| CPU microcode | `amd-ucode` (use `intel-ucode` on Intel) |
| Secondary data drive | `/dev/sda1` (ext4, mounted at `/mnt/Lemon_HDD`) |

The live session in these notes used **fish**, so variables are set with `set NAME value`. In bash, use `NAME=value` instead.

## Table of contents

1. [Partition and format](#1-partition-and-format)
2. [Clock and old boot entries](#2-clock-and-old-boot-entries)
3. [Btrfs subvolumes and mounts](#3-btrfs-subvolumes-and-mounts)
4. [Install the base system](#4-install-the-base-system)
5. [Configure the system in the chroot](#5-configure-the-system-in-the-chroot)
6. [Networking](#6-networking)
7. [mkinitcpio and pacman.conf](#7-mkinitcpio-and-pacmanconf)
8. [Add the CachyOS repositories](#8-add-the-cachyos-repositories)
9. [Apply CachyOS-Settings](#9-apply-cachyos-settings)
10. [Kernel command line, UKI and Secure Boot](#10-kernel-command-line-uki-and-secure-boot)
11. [Users, services and extra mounts](#11-users-services-and-extra-mounts)
12. [fstab and reboot](#12-fstab-and-reboot)
13. [Tips and gotchas](#13-tips-and-gotchas)

---

## 1. Partition and format

Boot the CachyOS ISO, open a root shell, and set some variables:

```fish
set DISK /dev/nvme0n1
set ESP  /dev/nvme0n1p1
set ROOT /dev/nvme0n1p2
```

Wipe the old partition table, then create the partitions with `cfdisk`:

```bash
sgdisk --zap-all $DISK
cfdisk $DISK
lsblk
```

In `cfdisk`, create a GPT layout:

| Partition | Size | Type |
| --- | --- | --- |
| `nvme0n1p1` | 512 MiB | EFI System |
| `nvme0n1p2` | remainder | Linux filesystem |

Format them:

```bash
mkfs.vfat -F32 -n ESP $ESP
mkfs.btrfs -f -L rootfs $ROOT
```

## 2. Clock and old boot entries

```bash
timedatectl set-timezone Indian/Maldives
timedatectl set-ntp true
```

**Optional:** clear out stale UEFI boot entries from previous installs. List them, then delete by number:

```bash
efibootmgr
efibootmgr -b 0000 -B    # repeat for each stale entry
```

> Only delete entries you recognise as old. You will create a fresh entry for this install in [step 10](#10-kernel-command-line-uki-and-secure-boot).

## 3. Btrfs subvolumes and mounts

Mount the raw Btrfs volume and create the subvolume layout. `AI` and `Games` get `chattr +C` (No-CoW), which suits large, frequently rewritten files:

```bash
mount $ROOT /mnt

btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@/home
btrfs subvolume create /mnt/@/var

mkdir /mnt/@/home/lemon
btrfs subvolume create /mnt/@/home/lemon/AI
btrfs subvolume create /mnt/@/home/lemon/Games

chattr +C /mnt/@/home/lemon/AI
chattr +C /mnt/@/home/lemon/Games
```

Now remount everything with the real options. `@` goes on `/mnt` first, then `@/home` and `@/var`, then the ESP:

```fish
set SUB_OPTS "noatime,space_cache=v2,compress=zstd:1,flushoncommit"

mount    -o $SUB_OPTS,subvol=@      $ROOT /mnt
mount -m -o $SUB_OPTS,subvol=@/home $ROOT /mnt/home/
mount -m -o $SUB_OPTS,subvol=@/var  $ROOT /mnt/var
mount -m -o umask=0077 $ESP /mnt/efi
```

> The ESP is mounted at `/efi`, not `/boot`, because the UKI is written to `/efi/EFI/Linux/`.

## 4. Install the base system

Refresh the mirrors from the live ISO, then `pacstrap`:

```bash
cachyos-rate-mirrors

pacstrap -K /mnt base base-devel linux-cachyos linux-firmware man-db vim \
  dosfstools e2fsprogs btrfs-progs util-linux arch-install-scripts \
  zram-generator efibootmgr pacman-contrib amd-ucode
```

Enter the chroot:

```bash
arch-chroot -S /mnt
export PS1="(ch) ${PS1}"
```

The `(ch)` prompt prefix is just a reminder that you are inside the chroot.

## 5. Configure the system in the chroot

### Time

```bash
ln -sf /usr/share/zoneinfo/Indian/Maldives /etc/localtime
hwclock --systohc
```

### Locales

Uncomment the locales you want in `/etc/locale.gen` (here `en_US.UTF-8` and `ja_JP.UTF-8`), then generate them:

```bash
vim /etc/locale.gen
locale-gen
```

### First-boot settings

```bash
systemd-firstboot --prompt
```

Answer the prompts: system locale `en_US.UTF-8`, message locale `en_US.UTF-8`, keymap `us`, hostname `miraigadget`.

> Inside a chroot you will see errors such as *"System has not been booted with systemd as init system"* and *"Failed to connect to systemd-hostnamed"*. They are expected here. The tool falls back to writing `/etc/locale.conf`, `/etc/vconsole.conf` and `/etc/hostname` directly, which is all you need.

Leave the chroot to link `resolv.conf`, then go back in:

```bash
exit
ln -sf /run/systemd/resolve/stub-resolv.conf /mnt/etc/resolv.conf
arch-chroot -S /mnt
export PS1="(ch) ${PS1}"
```

## 6. Networking

Wired DHCP with `systemd-networkd`. Create `/etc/systemd/network/20-ethernet.network`:

```ini
[Match]
Name=en*
Name=eth*

[Network]
DHCP=yes
IPv6PrivacyExtensions=yes
```

Enable the services:

```bash
systemctl enable systemd-resolved systemd-networkd
```

> This only covers Ethernet. Wi-Fi is not configured by this guide.

## 7. mkinitcpio and pacman.conf

### `/etc/mkinitcpio.conf`

A systemd-based initramfs. The `HOOKS` line used:

```bash
HOOKS=(base systemd autodetect microcode modconf kms keyboard consolefont block filesystems fsck)
```

> **Tip:** with the `systemd` hook, the Arch ecosystem generally recommends `sd-vconsole` rather than `consolefont` (it replaces both `keymap` and `consolefont` and loads the settings from `/etc/vconsole.conf`). If the early-boot console font doesn't apply, swap `consolefont` for `sd-vconsole`, then run `mkinitcpio -P` again.

### `/etc/pacman.conf`

Settings in my file:

```ini
[options]
Color
ILoveCandy
CheckSpace
ParallelDownloads = 5

[core]
Include = /etc/pacman.d/mirrorlist

[extra]
Include = /etc/pacman.d/mirrorlist

[multilib]
Include = /etc/pacman.d/mirrorlist
```

`[multilib]` is uncommented for 32-bit applications (Steam, Wine and similar). The CachyOS repos are added in the next step.

## 8. Add the CachyOS repositories

This is CachyOS's official script. It installs the CachyOS keyring and mirrorlists, detects your CPU's supported architecture level (x86-64-v3, v4, znver4 and so on), and edits `pacman.conf` accordingly.

```bash
curl https://mirror.cachyos.org/cachyos-repo.tar.xz -o cachyos-repo.tar.xz
tar xvf cachyos-repo.tar.xz && cd cachyos-repo
./cachyos-repo.sh
```

Clean up:

```bash
cd /
rm -rf cachyos-repo cachyos-repo.tar.xz
```

## 9. Apply CachyOS-Settings

Clone the settings repo and copy the pieces you want. This is a manual, selective copy of what a CachyOS install normally ships:

```bash
cd /tmp
pacman -Syy git
git clone https://github.com/CachyOS/CachyOS-Settings.git
cd CachyOS-Settings
```

```bash
# limits and helper scripts
cp -r etc/security/limits.d /etc/security/
cp usr/bin/game-performance usr/bin/kerver usr/bin/pci-latency usr/bin/topmem usr/bin/zink-run /usr/bin/
chmod +x /usr/bin/game-performance /usr/bin/kerver /usr/bin/pci-latency /usr/bin/topmem /usr/bin/zink-run

# kernel modules and sysctl
cp usr/lib/modprobe.d/amdgpu.conf    /etc/modprobe.d/
cp usr/lib/modprobe.d/blacklist.conf /etc/modprobe.d/
cp usr/lib/modules-load.d/ntsync.conf /etc/modules-load.d/
cp usr/lib/sysctl.d/70-cachyos-settings.conf /etc/sysctl.d/

# systemd drop-ins
cp -r usr/lib/systemd/journald.conf.d       /etc/systemd/
cp -r usr/lib/systemd/system-generators     /etc/systemd/
cp -r usr/lib/systemd/system.conf.d         /etc/systemd/
cp -r usr/lib/systemd/system                /etc/systemd/
cp -r usr/lib/systemd/timesyncd.conf.d      /etc/systemd/
cp -r usr/lib/systemd/user.conf.d           /etc/systemd/
cp -r usr/lib/systemd/user/session.slice.d  /etc/systemd/user/
cp usr/lib/systemd/zram-generator.conf      /etc/systemd/

# tmpfiles and udev
cp -r usr/lib/tmpfiles.d/*   /etc/tmpfiles.d/
cp -r usr/lib/udev/rules.d/* /etc/udev/rules.d/
cp -r usr/lib/udev/audio-pm-ctl /etc/udev/
```

Remove the pieces that don't apply to this machine:

```bash
rm -rf /etc/systemd/system/cachyos-iw-set-regdomain.*
rm -rf /etc/udev/rules.d/71-nvidia.rules
rm -rf /etc/udev/rules.d/85-iw-regulatory.rules
```

Install the supporting packages and enable the services:

```bash
pacman -S ananicy-cpp cachyos-ananicy-rules libluv lua-luv power-profiles-daemon
cd /
systemctl enable ananicy-cpp systemd-resolved systemd-oomd
```

> `amdgpu.conf` is AMD-specific. Skip it, and keep or drop the NVIDIA rule, to suit your own hardware.

## 10. Kernel command line, UKI and Secure Boot

### Kernel command line

Find your root partition's UUID:

```bash
blkid /dev/nvme0n1p2
```

Write `/etc/kernel/cmdline` (replace `<ROOT-UUID>` with the value from `blkid`):

```text
root=UUID=<ROOT-UUID> rootfstype=btrfs rootflags=subvol=/@ rw zswap.enabled=0
```

`rootflags=subvol=/@` tells the kernel which subvolume is `/`. `zswap` is disabled because swap is handled by zram (`zram-generator` is installed and CachyOS's `zram-generator.conf` was copied in step 9).

### mkinitcpio preset: UKI only

Edit `/etc/mkinitcpio.d/linux-cachyos.preset` so it builds a single UKI and no separate initramfs images:

```bash
ALL_kver="/boot/vmlinuz-linux-cachyos"

PRESETS=('default')
#PRESETS=('default' 'fallback')

default_uki="/efi/EFI/Linux/arch-linux-cachyos.efi"
default_options="--splash /usr/share/systemd/bootctl/splash-arch.bmp"
```

The `fallback` preset is left commented out, so **no fallback UKI is built**. Keep a live USB around in case an update breaks booting.

### Console font and build the UKI

```bash
mkdir -p /efi/EFI/Linux
pacman -S terminus-font
echo "FONT=ter-124b" >> /etc/vconsole.conf
mkinitcpio -P
```

### Secure Boot with sbctl

```bash
pacman -S sbctl
sbctl create-keys
sbctl enroll-keys --microsoft
sbctl sign --save /efi/EFI/Linux/arch-linux-cachyos.efi
sbctl verify
```

- `sbctl enroll-keys` normally needs the firmware to be in **Secure Boot Setup Mode** (clear or reset the Secure Boot keys in your UEFI settings first).
- `--microsoft` also enrolls Microsoft's keys, which keeps option ROMs and other signed firmware working.
- `--save` records the file in sbctl's database so it can be re-signed automatically after kernel updates.
- `sbctl verify` should list the UKI as signed.

### Create the UEFI boot entry

```bash
efibootmgr --create --disk /dev/nvme0n1 --part 1 \
  --label "Miraigadget-cachyos" \
  --loader 'EFI\Linux\arch-linux-cachyos.efi' --unicode
```

## 11. Users, services and extra mounts

### Root password and user

```bash
passwd
useradd -mG wheel,audio,video,render,users,optical lemon
```

`useradd` warns that `/home/lemon` already exists, because it was created in step 3 for the subvolumes. In that case it does not copy `/etc/skel`, so copy it yourself:

```bash
cp -a /etc/skel/. /home/lemon/
passwd lemon
```

### Services

```bash
systemctl enable systemd-timesyncd
systemctl enable paccache.timer
```

### Mount the `AI` and `Games` subvolumes

```bash
mount -o noatime,subvol=@/home/lemon/AI    /dev/nvme0n1p2 /home/lemon/AI
mount -o noatime,subvol=@/home/lemon/Games /dev/nvme0n1p2 /home/lemon/Games

chown -R lemon:lemon /home/lemon
lsblk
```

Run the `chown` **after** the mounts. Freshly mounted subvolumes are owned by root.

### Optional: secondary data drive

```bash
mount -m -o noatime,data=ordered,commit=120 /dev/sda1 /mnt/Lemon_HDD
```

## 12. fstab and reboot

Generate `fstab` from everything currently mounted:

```bash
genfstab -U / >> /etc/fstab
cat /etc/fstab
```

Check that it contains `/`, `/home`, `/var`, `/home/lemon/AI`, `/home/lemon/Games`, `/efi`, and `/mnt/Lemon_HDD` if you mounted it. All the Btrfs entries point at the same filesystem UUID and differ only in `subvol=`.

Leave the chroot, unmount, and reboot into the firmware settings to check the boot entry and Secure Boot state:

```bash
exit
umount -lnR /mnt
systemctl reboot --firmware-setup
```

After the first boot, confirm everything is in order:

```bash
sbctl status
sbctl verify
```

## 13. Tips and gotchas

- **Fish vs bash:** `set VAR value` is fish. In bash use `VAR=value`.
- **Chroot noise:** the `systemd-firstboot` PID 1 and `hostnamed` errors are harmless inside a chroot.
- **Mount order matters:** mount `@` first, then `@/home`, `@/var` and the ESP. `mount -m` creates the target directory if it is missing.
- **Don't skip `chown`:** do it after mounting `AI` and `Games`, or the mounts stay root-owned.
- **No fallback UKI:** the preset only builds `default`. If an update breaks your UKI, boot the live ISO, mount the system, `arch-chroot` in, and rebuild with `mkinitcpio -P`.
- **After kernel updates:** run `sbctl verify`. If the UKI shows as unsigned, run `sbctl sign-all`.
- **Intel instead of AMD:** swap `amd-ucode` for `intel-ucode` in `pacstrap`, and skip `amdgpu.conf` from CachyOS-Settings.

## Not covered by these notes

These steps were not part of my install log, so you will want to add them yourself:

- `sudo` access for the `wheel` group (`visudo`)
- Wi-Fi / NetworkManager
- A desktop environment or window manager, graphics drivers and audio

## References

- [Arch Linux Installation guide](https://wiki.archlinux.org/title/Installation_guide)
- [Arch Wiki: Unified kernel image](https://wiki.archlinux.org/title/Unified_kernel_image)
- [Arch Wiki: mkinitcpio](https://wiki.archlinux.org/title/Mkinitcpio)
- [Arch Wiki: Secure Boot (sbctl)](https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface/Secure_Boot)
- [CachyOS: adding the CachyOS repositories](https://wiki.cachyos.org/cachyos_repositories/how_to_add_cachyos_repo)
- [CachyOS-Settings](https://github.com/CachyOS/CachyOS-Settings)
