# Arch Linux Installation 2021

I had issues installing Arch onto Dell PowerEdge R620 with Hardware Raid 1 (UEFI) so these are the steps I took to successfully install and launch Arch Linux.

Download Arch Linux from https://archlinux.org/download/ and DD it onto your USB

## Installation
```bash
ping -c 3 google.com
```

```bash
timedatectl set-ntp true
```

Check Loaded Disks (Virtual or Physical)
```bash
fdisk -l
```

Partition Disk for Boot (EFI) and Root (Rest of the disk)
```bash
cfdisk /dev/sdX

# Select Label Type: GPT

# 512M Partition /mnt/efi TYPE=EFI
# ALL Partition /mnt TYPE=LINUX_86_64
```

```bash
mkfs.fat -F32 /dev/sdX1
mkfs.ext4 /dev/sdX2
```
```bash
mount /dev/sdX2 /mnt
mkdir /mnt/efi
mount /dev/sdX1 /mnt/efi
```

Installs linux kernel, base dependencies and some developer tools
```bash
pacstrap /mnt base base-devel linux linux-firmware vim
```

```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

##  Chroot Into The System
```bash
arch-chroot /mnt
```

Install additional software/dependencies to Arch Itself (on disk)
```bash
# efibootmgr is required for UEFI Boot/Grub installation!
pacman -S networkmanager alacritty zsh zsh-completions grub efibootmgr vi wget neofetch htop openssh sysstat
```

```bash
systemctl enable NetworkManager
```

Install GRUB into /EFI partition
```bash
grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=GRUB
grub-mkconfig -o /boot/grub/grub.cfg
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

## Setup Zsh Shell

```bash
zsh
```

```bash
# You should now see zsh-newuser-install, which will walk you through some basic configuration. If you want to skip this, press q. If you did not see it, you can invoke it manually with:
# autoload -Uz zsh-newuser-install
# zsh-newuser-install -f
# Make sure your terminal's size is at least 72أ—15 otherwise zsh-newuser-install will not run.
```

Making Zsh your default shell
```bash
chsh -l
chsh -s /usr/bin/zsh
# If you are using systemd-homed, run:
# homectl update --shell=/usr/bin/zsh user
# Tip: If replacing bash, users may want to move some code from ~/.bashrc to ~/.zshrc (e.g. the prompt and the aliases) and from ~/.bash_profile to ~/.zprofile (e.g. the code that starts the X Window System).
```

## Setup Xorg & Window Tiling
```bash
sudo pacman -S git i3 i3-gaps xorg xorg-xinit
# 1 3 4 5 (i3-gaps - gives spacing)
```

## Install Bitmap Fonts

```bash
git clone https://github.com/Tecate/bitmap-fonts.git
cd bitmap-fonts
sudo cp -avr bitmap/ /usr/share/fonts/
cd /usr/share/fonts/bitmap
mkfontscale
mkfontdir
xset fp+ /usr/share/fonts/bitmap
fc-cache -fv
```
