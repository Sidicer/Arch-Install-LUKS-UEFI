# Arch Linux Installation
UEFI, with full disk encryption (LUKS) and LVM for flexibility.

---

## Prepare your bootable USB drive
[Download](https://archlinux.org/download) Arch Linux and `dd` it onto your USB drive
```sh
# /dev/sdX - replace with your USB Flash Drive letter (lsblk; fdisk -l)
sudo dd bs=4M if=/archlinux-2024.04.01-x86_64.iso of=/dev/sdX && sync
```

---

# Installation

Check if you have network connectivity and sync your system clock
```sh
ping -c 3 google.com
```

```sh
timedatectl set-ntp true
```

## Disk partitioning

Confirm your storage layout and note the drive you will be installing to
```sh
fdisk -l # or
lsblk
# /dev/nvme0n1
```

Partition the drive to have a 1GB partition for boot and then the rest do Linux Filesystem (as we will be encrypting it)
```sh
cfdisk /dev/nvme0n1

# Select Label Type: GPT
# 1G Partition /boot TYPE=EFI
# 100%FREE /root TYPE=LINUX_86_64
```

You should now have two `/dev/nvme0n1p1` & `/dev/nvme0n1p2` partitions

Format the first `1G` partition to `VFAT32`
```sh
mkfs.vfat -F32 /dev/nvme0n1p1
```
And now we can encrypt and setup the our partition

## Disk Encryption

Encrypt the full partition
```sh
cryptsetup luksFormat /dev/nvme0n1p2
# You will have to type "YES" to confirm formatting
```

After that succeeds we can open that encrypted partition to work with it
```sh
cryptsetup luksOpen /dev/nvme0n1p2 cryptroot
# You can change "cryptroot" to whatever you like, but you will have to
# remember and use your name instead of cryptroot for the rest of the install
```

## LVM Creation

A logical volume needs a volume group which in turn needs a physical volume. So lets set those up
```sh
# Create your physical volume
pvcreate /dev/mapper/cryptroot

# Create a volume group (I will call it "vg0")
vgcreate vg0 /dev/mapper/cryptroot

# Create the logical volumes (root, home, swap)
# Notice -L and -l, one is for fixed size, the other is percentage
lvcreate -L 32G vg0 -n swap # If you plan to use hybernation - set the same size as your RAM
lvcreate -L 120G vg0 -n root # Modify "120G" to what ever size you think fits your root setup
lvcreate -l 100%FREE vg0 -n home # Fill the rest of the volume group for home
```

Format and mount the newly created volumes
```sh
mkfs.ext4 /dev/mapper/vg0-root
mkfs.ext4 /dev/mapper/vg0-home
mkswap /dev/mapper/vg0-swap

mount /dev/mapper/vg0-root /mnt

mkdir /mnt/home
mount /dev/mapper/vg0-home /mnt/home

mkdir /mnt/boot
mount /dev/nvme0n1p1 /mnt/boot

swapon -s /dev/mapper/vg0-swap
```

## Install the system

Installs linux kernel, base dependencies and text editor
```sh
pacstrap -i /mnt base base-devel linux linux-firmware lvm2 vim
```

I usually install other required packages now rather than after chroot'ing into the system
```sh
pacstrap -i /mnt networkmanager zsh git curl openssh sysstat intel-ucode
# If you're on AMD replace "intel-ucode" with "amd-ucode"
```

## Generate fstab

```sh
genfstab -U /mnt >> /mnt/etc/fstab
```

Check if swap was written also
```sh
cat /mnt/etc/fstab
```
And if not find your `vg0-swap` UUID with `blkid /dev/mapper/vg0-swap` and add it at the end of the fstab
```cfg
UUID=SWAP_UUID none swap defaults 0 0
```

##  Chroot into your new system
```sh
arch-chroot /mnt
```

### Setup the bootloader (`systemd-boot`)

```sh
bootctl --path=/boot install
```

Get the partition UUID which the bootloader will need to load (it should be the partition you encrypted and not the actual LVM)
```
# We write it to a file to have it on hand when writing the bootloader entry
blkid /dev/nvme0n1p2 > /boot/loader/entries/arch.conf
```

Edit the entry file and add the required info
```sh
# vim /boot/loader/entries/arch.conf
# replace intel-ucode with amd-ucode if AMD
# replace PARTITION_ID with the UUID that we entered here with blkid in the previous step
```
```
title Arch Linux
linux /vmlinuz-linux
initrd /intel-ucode.img
initrd /initramfs-linux.img
options cryptdevice=UUID=PARTITION_ID:vg0 root=/dev/mapper/vg0-root quiet splash rw
```

Save and exit with `:wq` and update bootloader
```sh
bootctl update
```

## TODO: Update Mkinitpcio ADD INFO HERE

Enable NetworkManager
```sh
systemctl enable NetworkManager
```

Change your region and localtime, sync clock
```bash
ln -sf /usr/share/zoneinfo/REGION/CITY /etc/localtime
hwclock --systohc
```

```bash
# Edit /etc/locale.gen and uncomment en_US.UTF-8 UTF-8 and other needed locales. Generate the locales by running:
locale-gen

echo "LANG=en_US.UTF-8" > /etc/locale.conf
```

Setup your machines hostname
```bash
echo "Machines Hostname" > /etc/hostname
```

Setup root password:
```bash
passwd
```

Exit chroot, unmount partitions and reboot
```bash
exit #(ctrl+d)
umount -R /mnt
reboot
```

## After Booting up

Just in case, update everything:
```bash
pacman -Syy
pacman -Syu
```

Create another user (DO NOT USE ROOT FOR DAILY USE!)
```bash
visudo
# Find where it says "root ALL=(ALL) ALL".
# Type "o" to insert a new line below it.
# Now type what you want to insert, eg "username ALL=(ALL) ALL".
# Hit esc to exit insert-mode.
# Type ":x" to save and exit.
```
```bash
useradd -m -g users -G wheel -s /bin/bash USERNAME
passwd USERNAME
```

## Setup SSH

```bash
# Whenever changing the configuration, use sshd in test mode before restarting the service to ensure it will be able to start cleanly. Valid configurations produce no output.
# use: sshd -t
```
Setup SSH Welcome Banner:
```bash
sudo vim /etc/ssh/sshd_config
# Uncomment # Banner /etc/issue
# :wq
```
```bash
sudo vim /etc/issue
# Add a welcome message
# :wq
```
```bash
sudo systemctl start sshd
sudo systemctl enable sshd
```
