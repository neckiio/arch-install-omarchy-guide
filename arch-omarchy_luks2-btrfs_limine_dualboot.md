# Arch Linux Installation with LUKS2, BTRFS, Limine, and Omarchy  
### Dual-Boot Setup with Windows 11

## Abstract

This document describes a manual installation of Arch Linux on a UEFI system using a **LUKS2-encrypted BTRFS root filesystem**, integrated with the **Limine bootloader**, and configured in a **dual-boot environment with Windows 11**.

The installation is performed entirely from the official Arch Linux ISO and follows a traditional, step-by-step approach without automated installers. After the base system is bootable, **Omarchy** is installed from the console on the freshly installed system.

The guide emphasizes clarity, control, and long-term maintainability. It is intended for users who prefer understanding each stage of the installation process, particularly disk layout, encryption, subvolume design, and bootloader integration, while preserving an existing Windows setup.

## Table of Contents

## Table of Contents

- [Introduction](#introduction)
- [Preparation](#preparation)
- [Step 1 — Set keymap and console font](#step-1--set-keymap-and-console-font)
- [Step 2 — Prepare the environment](#step-2--prepare-the-environment)
  - [Ensure network connectivity](#ensure-network-connectivity)
- [Step 3 — Enable network time synchronization](#step-3--enable-network-time-synchronization)
- [Step 4 — Disk layout overview](#step-4--disk-layout-overview)
- [Step 5 — Create Linux and swap partitions](#step-5--create-linux-and-swap-partitions)
- [Step 6 — Encrypt the Linux partition with LUKS2](#step-6--encrypt-the-linux-partition-with-luks2)
  - [Initialize LUKS2 encryption](#initialize-luks2-encryption)
- [Step 7 — Format encrypted partition as BTRFS and create subvolumes](#step-7--format-encrypted-partition-as-btrfs-and-create-subvolumes)
  - [Format the encrypted partition as BTRFS](#format-the-encrypted-partition-as-btrfs)
  - [Mount the filesystem to create subvolumes](#mount-the-filesystem-to-create-subvolumes)
  - [Format and enable swap](#format-and-enable-swap)
  - [Mount ESP](#mount-esp)
- [Step 8 — Install the Arch Linux base system](#step-8--install-the-arch-linux-base-system)
  - [Synchronize package databases](#synchronize-package-databases)
  - [Install base system and essential packages](#install-base-system-and-essential-packages)
  - [Generate filesystem table (fstab)](#generate-filesystem-table-fstab)
- [Step 9 — Chroot into the new system and perform base configuration](#step-9--chroot-into-the-new-system-and-perform-base-configuration)
  - [Set timezone](#set-timezone)
  - [Configure locale](#configure-locale)
  - [Configure keyboard layout (TTY)](#configure-keyboard-layout-tty)
  - [Set hostname](#set-hostname)
  - [Set root password](#set-root-password)
- [Step 10 — Configure mkinitcpio for LUKS2 and BTRFS](#step-10--configure-mkinitcpio-for-luks2-and-btrfs)
  - [Configure HOOKS](#configure-hooks)
  - [Regenerate initramfs](#regenerate-initramfs)
- [Step 11 — Install and configure the Limine bootloader (UEFI)](#step-11--install-and-configure-the-limine-bootloader-uefi)
  - [Install Limine files](#install-limine-files)
  - [Create Limine configuration](#create-limine-configuration)
  - [Register Limine in UEFI](#register-limine-in-uefi)
- [Step 12 — First boot and initial login](#step-12--first-boot-and-initial-login)
- [Step 13 — Enable essential system services](#step-13--enable-essential-system-services)
  - [Enable networking](#enable-networking)
- [Step 14 — Update system and prepare for Omarchy installation](#step-14--update-system-and-prepare-for-omarchy-installation)
  - [Ensure internet connectivity](#ensure-internet-connectivity)
  - [Update system packages](#update-system-packages)
- [Step 15 — Install Omarchy from the console](#step-15--install-omarchy-from-the-console)
- [Step 16: Restore Limine menu for dual-boot (post-Omarchy)](#step-16-restore-limine-menu-for-dual-boot-post-omarchy)


## Introduction

*This document does not replace the official Arch Linux installation guide and the accompanying documentation. It is merely a condensed note of the most important steps, intended for my personal use. Use it at your own risk, and keep in mind that some of the instructions below may be outdated or unsuitable for your specific case.*

This guide describes a manual Arch Linux installation using a **LUKS2-encrypted BTRFS system partition**, **Limine bootloader integration**, and a **dual-boot setup with Windows 11**. In addition, **Omarchy** will be installed **after the first reboot**, directly from the console on the freshly installed Arch system.

All installation steps are performed from the official **Arch Linux ISO**. The system is brought to a bootable state first, then Omarchy is added on top of the base installation, ensuring full control over encryption, subvolumes, boot entries, and session management.

This guide prioritizes **clarity and reproducibility over convenience**. Each command is intentional and assumes the reader understands UEFI booting, disk partitioning, and Linux system administration. The objective is a clean, maintainable system that coexists correctly with an existing Windows EFI setup, without altering Windows boot components.

Secure Boot is not covered. The focus is on correctness, transparency, and long-term stability.

## Preparation

- A basic familiarity with manual Linux installations is assumed (e.g. Arch Linux, Kali Linux, or similar distributions).

- Windows 11 is already installed and working on the target system.

- A dedicated partition for Omarchy (Arch Linux) has been created in advance, left unformatted and available as free disk space (for example, created using tools such as Partition Wizard).

- The Arch Linux ISO has been downloaded from the official Arch Linux website.

- The Arch Linux ISO has been written to a bootable USB drive (using Ventoy or any equivalent tool).

- The system has been successfully booted from the official Arch Linux ISO.

- The system firmware has been verified to be UEFI, using:

```sh
cat /sys/firmware/efi/fw_platform_size
```

## Step 1: Set keymap and console font

Set the keyboard layout for the current live session. This affects input in the console during installation only.

List available keymaps:

```sh
localectl list-keymaps
```

Load the desired keymap (example for español):

```sh
loadkeys es
```

Optionally, set a console font for better readability, especially on high-resolution displays.

List available console fonts:

```sh
ls /usr/share/kbd/consolefonts/
```

Load a console font (example):

```sh
setfont ter-132n
```

## Step 2 — Prepare the environment

### Ensure network connectivity

This guide assumes a laptop is being used and that Internet access will be established over a wireless connection using **iwd**.

First, start the interactive iwd shell:

```sh
iwctl
```

List available wireless devices and identify the correct interface:

```sh
device list
```

Scan for available wireless networks:

```sh
station <device> scan
```

List detected networks:

```sh
station <device> get-networks
```

Connect to the desired network:

```sh
station <device> connect <SSID>
```

Exit the iwd shell once the connection is established:

```sh
exit
```

Verify network connectivity by sending ICMP requests to a known host:

```sh
ping -c 3 archlinux.org
```

## Step 3 — Enable network time synchronization

Ensure that system time is synchronized over the network using **systemd-timesyncd**.

Enable NTP synchronization:

```sh
timedatectl set-ntp true
```

Verify the current time and synchronization status:

```sh
timedatectl status
```

The system clock must be accurate before proceeding, especially for package installation and encryption-related operations.

## Step 4 — Disk layout overview

Before partitioning and encryption, it is important to define and understand the disk layout that will be used.

This guide assumes a **UEFI system with an existing Windows 11 installation**, and a disk layout similar to the following:

- **EFI System Partition (ESP)**  
  - Already exists  
  - Used by Windows Boot Manager and Limine (It is usually *nvme0n1p1*)  
  - *Not formatted*

- **Microsoft Reserved / Windows partitions**  
  - Already exist  
  - Left untouched

- **Linux system partition (new)**  
  - Dedicated partition for Arch Linux  
  - Will be encrypted using **LUKS2**  
  - Formatted as **BTRFS**  
  - Will contain all Linux subvolumes

No changes are made to the existing Windows partitions or their EFI entries. The Linux installation will reuse the existing ESP and coexist with Windows using separate boot entries.

At this stage, **only the layout is defined**. Actual partitioning, encryption, and formatting will be performed in the next steps.

## Step 5 — Create Linux and swap partitions

At this stage, new partitions will be created for the Linux system and swap space.  
Only the previously reserved free space will be used.

> ⚠️ This step is destructive. Double-check the selected disk before proceeding.

List available block devices and identify the target disk:

```sh
lsblk
```

Launch the partitioning tool:

```sh
cfdisk /dev/<disk>
```

*Inside **cfdisk***, perform the following actions:

1. Select the __*free space*__ reserved for **Linux**.

2. Create a **swap partition** (**new**):

- Size: 5G

- Type (**Type**): Linux swap

3. Create a **Linux system partition** using the remaining free space:

- Type (**Type**): Linux filesystem

4. Write the partition table and confirm the changes (**Write** and **YES**).

5. Exit **cfdisk**. (**quit**)

After exiting, verify the new partition layout:

```sh
lsblk
```

At the end of this step, the disk should contain:

One **swap partition** (5 GiB)

One **Linux partition** (to be encrypted and formatted as BTRFS in the next step)
 
## Step 6 — Encrypt the Linux partition with LUKS2

In this step, the Linux system partition created previously will be encrypted using **LUKS2**.  
All Linux data, including BTRFS subvolumes, will reside inside this encrypted container.

> ⚠️ This operation is destructive. Ensure the correct partition is selected.

### Initialize LUKS2 encryption

Encrypt the Linux partition:

```sh
cryptsetup luksFormat /dev/<linux-partition>
```

for example (Replace **x** with the actual partition number where the **Linux system** will be installed.)

```sh
cryptsetup luksFormat /dev/nvme0n1px
```

You will be prompted to confirm the operation and set a passphrase.
Choose a strong passphrase and remember it — it will be required at every boot.

It should be more than obvious that the password used to encrypt this system partition must not be random and obvious, but it also must not be forgotten.

Later, the UUID of this container will be needed, which can be obtained using the command cryptsetup luksUUID /dev/nvme0n1px or the command ls -l --time-style=+ /dev/disk/by-uuid/. Make sure to save this UUID somewhere.

Open the encrypted container

Unlock and map the encrypted partition:

```sh
cryptsetup open /dev/<linux-partition> cryptroot
```

This creates a decrypted device at:

```sh
/dev/mapper/cryptroot
```

Verify that the encrypted device is active:

```sh
lsblk
```

At this point, the encrypted container is ready to be formatted with BTRFS.
Subvolume creation and mounting will be handled in the next step.

## Step 7 — Format encrypted partition as BTRFS and create subvolumes

With the encrypted container open at `/dev/mapper/cryptroot`, the filesystem will now be created and structured using **BTRFS subvolumes**.

### Format the encrypted partition as BTRFS

```sh
mkfs.btrfs /dev/mapper/cryptroot
```

### Mount the filesystem to create subvolumes
Mount (*temporarily*) the newly created btrfs file system in \mnt:

```sh
mount /dev/mapper/cryptroot /mnt
```

And let's make some subvolumes.

```sh
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@var_log
btrfs subvolume create /mnt/@var_cache
```

> ⚠️ There is no need to create a subvolume for snapshots; Omarchy will take care of it during installation.

Verify the subvolumes:

```sh
btrfs subvolume list /mnt
```

Let's unmount the temporarily mounted file system and mount the subvolumes instead (incl. the the vfat ESP partition). If you don't want compression just remove the compress option, but level 1 is great for NMVE disks. Also noatime is fine for personal desktop setup.

Unmount the filesystem:

```sh
umount /mnt
```

Mount the filesystem:

```sh
mount -o noatime,compress=zstd:1,subvol=@ /dev/mapper/cryptroot /mnt
```

Create mount points:

```sh
mkdir -p /mnt/{home,var/log,var/cache}
```

Mount the remaining subvolumes:

```sh
mount -o noatime,compress=zstd:1,subvol=@home /dev/mapper/cryptroot /mnt/home
mount -o noatime,compress=zstd:1,subvol=@var_log /dev/mapper/cryptroot /mnt/var/log
mount -o noatime,compress=zstd:1,subvol=@var_cache /dev/mapper/cryptroot /mnt/var/cache
```

### Format and enable swap

Format the swap partition:

```sh
mkswap /dev/<swap-partition>
```

Enable swap:

```sh
swapon /dev/<swap-partition>
```

Verify swap status:

```sh
swapon --show
```

### mount ESP
Create mount points

```sh
mkdir -p /mnt/boot
```

mount the ESP partition (It is usually *nvme0n1p1*):

```sh
 mount /dev/<efi-system-partition> /mnt/boot 
```

At this point, the encrypted BTRFS filesystem and swap space are ready.
The system is now prepared for base package installation.

## Step 8 — Install the Arch Linux base system

With all filesystems mounted under `/mnt`, the base Arch Linux system can now be installed.

Many guides suggest that the following is sufficient:

```sh
pacstrap -K /mnt base linux linux-firmware
```

While technically correct, this guide installs a more complete and practical base system from the start, avoiding unnecessary rework later.

### Synchronize package databases

```sh
pacman -Syy
```

### Install base system and essential packages

```sh
pacstrap -K /mnt base base-devel linux linux-firmware git curl perl wget nano vim limine cmake make gcc btrfs-progs efibootmgr cryptsetup iwd networkmanager reflector bash-completion avahi acpi acpi_call acpid alsa-utils pipewire pipewire-alsa pipewire-pulse pipewire-jack wireplumber sof-firmware bluez bluez-utils cups docker util-linux terminus-font openssh sudo rsync intel-ucode
```

> ⚠️ Replace intel-ucode with amd-ucode if you are using an AMD processor.
> ⚠️ This process may take a considerable amount of time depending on your internet connection (do programmer things like go get a coffee xd)

This package set provides:

- A functional base system

- Networking (wired and wireless)

- Audio stack (PipeWire)

- Power and hardware support

- Printing, Bluetooth, and basic services

- Tools required for bootloader, encryption, and BTRFS management

### Generate filesystem table (fstab)

Generate the filesystem table based on the current mount layout:

```sh
genfstab -U /mnt >> /mnt/etc/fstab
```

Review the generated file and correct any obvious errors before continuing (To exit nano press Ctrl + X and to save press Ctrl + O followed by Enter.):

```sh
nano /mnt/etc/fstab
```

At this point, the base system is installed and ready for configuration inside the new environment.

## Step 9 — Chroot into the new system and perform base configuration

Change root into the newly installed system:

```sh
arch-chroot /mnt
```

At this point, all commands are executed inside the installed system, not the live ISO.

### Set timezone

Set the correct timezone (example: Mexico City):

```sh
ln -sf /usr/share/zoneinfo/America/Mexico_City /etc/localtime
```

Synchronize the hardware clock:

```sh
hwclock --systohc
```

### Configure locale

Edit the locale configuration file and uncomment the desired locale:

```sh
nano /etc/locale.gen
```

Example:

```sh
es_MX.UTF-8 UTF-8
```

Generate locales:

```sh
locale-gen
```

Set the system-wide locale:

```sh
echo "LANG=es_MX.UTF-8" > /etc/locale.conf
```

### Configure keyboard layout (TTY)

Set the keyboard layout for the console:

```sh
echo "KEYMAP=es" > /etc/vconsole.conf
```

Optionally, set a console font:

```sh
echo "FONT=ter-132n" >> /etc/vconsole.conf
```

### Set hostname

Set the system hostname:

```sh
echo "arch" > /etc/hostname
``` 

Configure hosts file:

```sh
nano /etc/hosts
```

Add the following:

```sh
127.0.0.1   localhost
::1         localhost
127.0.1.1   archlinux.localdomain arch
```

### Set root password

Set up the root password

```sh
passwd
```

Add a new user, add it to wheel group and set a password ( replace "name-user" with the name of your choice).

```sh
useradd -mG wheel <name-user>
passwd <name-user>
```

Execute the following command and uncomment the line to let members of group wheel execute any program

```sh
EDITOR=nano visudo
```

uncomment the line *%wheel ALL=(ALL:ALL) ALL*

At this stage, the system has basic identity, locale, and time configuration in place.
The next steps will focus on initramfs, encryption hooks, and bootloader configuration.

## Step 10 — Configure mkinitcpio for LUKS2 and BTRFS

The initramfs must be configured to unlock the encrypted root partition and mount the BTRFS filesystem at boot.

Edit the mkinitcpio configuration file:

```sh
nano /etc/mkinitcpio.conf
``` 

### Configure HOOKS

Locate the HOOKS line and modify it as follows (To exit nano press Ctrl + X and to save press Ctrl + O followed by Enter.):
*IT MUST contain EXACTLY something like this:*

```sh
HOOKS=(base udev autodetect keyboard keymap modconf block encrypt filesystems fsck)
```

Explanation of the relevant hooks:

- keyboard, keymap: ensure correct keyboard input at the LUKS passphrase prompt

- block: required for block device detection

- encrypt: unlocks the LUKS2 container

- filesystems: mounts the root filesystem

### Regenerate initramfs

After saving the file, regenerate the initramfs images:

```sh
mkinitcpio -P
```

Verify that the process completes without errors.

At this point, the system is capable of unlocking the encrypted root filesystem during early boot.

## Step 11 — Install and configure the Limine bootloader (UEFI)

This step installs **Limine** and configures it to boot an encrypted BTRFS root filesystem, while preserving the existing Windows 11 boot setup.

### Install Limine files

Create a directory for Limine:

```sh
mkdir -p /boot/EFI/limine
```

Copy Limine UEFI binaries:

```sh
cp /usr/share/limine/BOOTX64.EFI /boot/efi/limine/
cp /usr/share/limine/limine.cfg /boot/efi/limine/
```

Create a fallback boot entry (recommended):

```sh
mkdir -p /boot/efi/EFI/BOOT
cp /usr/share/limine/BOOTX64.EFI /boot/efi/EFI/BOOT/BOOTX64.EFI
```

### Create Limine configuration

Edit the Limine configuration file:

```sh
nano /boot/EFI/limine/limine.conf
```

Minimal configuration example:

```text
 timeout: 30

 /Windows 11
     protocol: efi_chainload
     image_path: boot():/EFI/Microsoft/Boot/bootmgfw.efi

 /Arch Linux
     protocol: linux
     path: boot():/vmlinuz-linux
     cmdline: cryptdevice=UUID=<device-UUID>:root root=/dev/mapper/cryptroot rw rootflags=subvol=@ rootfstype=btrfs
     module_path: boot():/initramfs-linux.img

 /Arch Linux (fallback)
     protocol: linux
     path: boot():/vmlinuz-linux
     cmdline: cryptdevice=UUID=<device-UUID>:root root=/dev/mapper/cryptroot rw rootflags=subvol=@ rootfstype=btrfs
     module_path: boot():/initramfs-linux-fallback.img
```

> ⚠️ Remember being advised to save an UUID? Now is the time to use it. Replace the <device-UUID> above with the UUID of your LUKS container.
For spacing in limine.conf, do not use tabs, use spaces, and respect the indentation.

### Register Limine in UEFI

Create a UEFI boot entry:

```sh
efibootmgr --create --disk /dev/<efi-system-partition> --part <efi-part-number> --label "Arch Limine" --loader '\EFI\limine\BOOTX64.EFI' --unicode
```

N.B. --disk /dev/nvme0n1 --part 1 means /dev/nvme0n1p1

N.B. Do NOT include boot when pointing to Limine Bootloader file.

N.B. In a Limine config, boot():\/ represents the partition on which limine.conf is located.

Verify boot entries:

```sh
efibootmgr -v
```

At this point, Limine is installed and configured to boot both Arch Linux and Windows 11.
It's time to exit the chroot, unmount the /mnt, close the crypted container and reboot to the newly installed Arch Linux.

```sh
exit
umount -R /mnt
swapoff /dev/<swap-partition>
cryptsetup close cryptroot
reboot
```

Do not forget to unplug the installation media (USB stick).

## Step 12 — First boot and initial login

After exiting the chroot environment, reboot the system:

```sh
reboot
```

During the first boot, the following sequence is expected:

1. LUKS passphrase prompt
The system will request the passphrase to unlock the encrypted root partition.
Enter the LUKS passphrase configured during encryption.

2. System boot process
Once the encrypted container is unlocked, the system will proceed with the normal boot sequence.

3. User login prompt
At the TTY login screen, enter the username created during installation.

4. User password prompt
Enter the corresponding user password to complete the login.

A successful login confirms that:

1. The encrypted root filesystem is unlocked correctly

2. The initramfs and bootloader configuration are valid

3. User authentication is functioning as expected

At this point, the base Arch Linux system is fully operational and ready for post-installation tasks.

## Step 13 — Enable essential system services

Before rebooting, required system services must be enabled so the system can operate correctly after boot.

### Enable networking

This guide installs both `iwd` and `NetworkManager`. NetworkManager will be used as the primary network manager.

Enable NetworkManager and others:

```sh
sudo systemctl enable NetworkManager
sudo systemctl enable iwd
sudo systemctl enable sshd
sudo systemctl enable systemd-networkd
sudo systemctl enable systemd-resolved
sudo systemctl enable systemd-timesyncd
sudo systemctl enable bluetooth
sudo systemctl enable cups
sudo systemctl enable avahi-daemon
sudo systemctl enable acpid
sudo systemctl enable reflector.timer
```

Enable audio services

PipeWire services are socket-activated and do not require manual enabling. No action is required at this stage.

Verify enabled services
Optionally verify enabled services:

```sh
systemctl list-unit-files --state=enabled
```

restart the laptop

```sh
reboot
```

## Step 14 — Update system and prepare for Omarchy installation

Before installing Omarchy, the system should be fully updated and equipped with the required build and download tools.

### Ensure internet connectivity

Make sure the system has a working internet connection. For example (if needed):

```sh
nmtui
nmcli device status
ping -c 3 archlinux.org
```

### Update system packages

Synchronize repositories and update the system:

```sh
sudo pacman -Syu
```

Backup current mirror list

Create a backup of the current mirror list:

```sh
sudo cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.bak
```

Generate a fresh mirror list

Use reflector to select fast HTTPS mirrors:

```sh
sudo reflector --verbose --latest 10 --protocol https --sort rate --save /etc/pacman.d/mirrorlist
```

Refresh package databases and update again:

```sh
sudo pacman -Syyu
```

## Step 15 — Install Omarchy from the console

Now that the base Arch Linux system is installed, encrypted, bootable, and you have logged in for the first time, the next step is to install **Omarchy**.

Run the official Omarchy installer script

Execute the installation script directly from the Omarchy website:

```sh
curl -fsSL https://omarchy.org/install | bash
```

This command downloads and pipes the installation script to bash, triggering an interactive installer that configures your system according to Omarchy defaults. 

During the installation:

You may be prompted to enter **sudo** and your user password.

You will be asked for your **name** and **email** for Git configuration (It's possible that he won't ask for them.).

The installer will fetch and install packages, set up configurations (including Hyprland, utilities, themes, and defaults), and adjust system settings for a polished desktop experience.

The process can take **several minutes** depending on your internet speed.

> ⚠️ You may need to restart the Omarchy installation more than five times.

Once the installation finishes, click **Reboot now** when prompted.

## Step 16: Restore Limine menu for dual-boot (post-Omarchy)

After rebooting into Omarchy, a terminal can be opened using:

```sh
SUPER + ENTER
```

Enter root mode (enter your password):

```sh
sudo -i
```

Detect existing boot entries

Scan for existing boot entries (including Windows):

```sh
limine-scan
```

A numbered list will be displayed.
Enter the number corresponding to Windows and press **Enter**.

Relocate Omarchy EFI binary

Before making changes, create a backup of the Omarchy EFI file:

```sh
cp /boot/EFI/Linux/omarchy-linux.efi /boot/EFI/Linux/omarchy-linux.efi.bak
```

Move the EFI binary to Limine’s directory:

```sh
mv /boot/EFI/Linux/omarchy-linux.efi /boot/EFI/limine/
``` 

Backup and edit Limine configuration

Backup the existing Limine configuration file:

```sh
mv /boot/EFI/limine/limine.conf /boot/EFI/limine/limine.conf.bak
```

Edit the active Limine configuration file:

```sh
nano /boot/limine.conf
```

Locate the IMAGE_PATH (or PATH) line for Omarchy and update it.

Original value:

```sh
/EFI/Linux/omarchy-linux.efi
```

Replace it with:

```sh
/EFI/limine/omarchy-linux.efi
```

Save the file and exit the editor.

Reboot and verify

Reboot the system:

```sh
reboot
```

On the next boot, Limine should display both Omarchy and Windows 11 as selectable entries.

# References
- GitHub: yovko/arch_linux_install_notes.md
- youtube: https://www.youtube.com/watch?v=kE0foLfYkfk&t=1065s
- youtube: https://www.youtube.com/watch?v=kXqk91R4RwU&t=315s
