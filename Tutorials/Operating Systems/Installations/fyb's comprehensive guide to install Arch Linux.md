# Overview

I have tried to install Arch Linux on a dual-booted, NVMe machine for 5 times. I succeeded in the 6th try.

That system was wiped by [Deven](https://trinity.moe)'s famous Zelda theme song script. I installed Arch for the 7th time and decided to document every step of it.

Any part of text formatted in *italics* are placeholders, which should be observed carefully and modified according to your situation, if necessary.

Many steps contain two different formats:
 - Commands that I used to install my system, they explicitly contain details that may be specific to my system
 - Commands that contain placeholders, they should be *translated* according to your system

## Table of Contents
 - [Creation of installation media](https://gist.github.com/fybalaban/d188fe0de48ea92c5b74715471e0f39b#creation-of-an-installation-media)
 - [Booting from USB](https://gist.github.com/fybalaban/d188fe0de48ea92c5b74715471e0f39b#booting-from-usb)
 - [First steps](https://gist.github.com/fybalaban/d188fe0de48ea92c5b74715471e0f39b#first-steps)
   - [Load keymap](https://gist.github.com/fybalaban/d188fe0de48ea92c5b74715471e0f39b#1-first-and-foremost-load-the-correct-keymap-using-loadkeys-command)
   - [Check boot mode](https://gist.github.com/fybalaban/d188fe0de48ea92c5b74715471e0f39b#2-check-boot-mode)
   - [Connect to Internet](https://gist.github.com/fybalaban/d188fe0de48ea92c5b74715471e0f39b#3-connect-to-the-internet)
   - [Set timezone](https://gist.github.com/fybalaban/d188fe0de48ea92c5b74715471e0f39b#4-set-timezone-and-sync-time-using-ntp)
 - [Creating partitions](https://gist.github.com/fybalaban/d188fe0de48ea92c5b74715471e0f39b#creating-partitions)
 - [Installing base system](https://gist.github.com/fybalaban/d188fe0de48ea92c5b74715471e0f39b#installing-base-system)
 - [Chrooting into system](https://gist.github.com/fybalaban/d188fe0de48ea92c5b74715471e0f39b#chrooting-into-system)
   - [Set timezone](https://gist.github.com/fybalaban/d188fe0de48ea92c5b74715471e0f39b#1-set-the-timezone)
   - [Set localization](https://gist.github.com/fybalaban/d188fe0de48ea92c5b74715471e0f39b#2-set-localization)
     - Generate locales
     - Set default vconsole (tty) keymap
     - Set default vconsole (tty) font
   - [Network configuration](https://gist.github.com/fybalaban/d188fe0de48ea92c5b74715471e0f39b#3-network-configuration)
   - [Add root password](https://gist.github.com/fybalaban/d188fe0de48ea92c5b74715471e0f39b#4-add-password-for-root-user)
   - [Install GRUB](https://gist.github.com/fybalaban/d188fe0de48ea92c5b74715471e0f39b#5-installing-the-grub-bootloader)
 - [First boot](https://gist.github.com/fybalaban/d188fe0de48ea92c5b74715471e0f39b#first-boot)

## Creation of an installation media

Download the latest ISO file from https://archlinux.org/download/.

I will be using an USB drive to create the bootable Arch media. Check [this Arch wiki article](https://wiki.archlinux.org/title/USB_flash_installation_medium) for details.

1. Find name of your USB drive using ```lsblk```. For example my USB drive is */dev/sda*
2. Make sure it's not mounted, if necessary unmount it
3. Run the following command, **do not** append any number to end of it

> \# cat *path/to/archlinux.iso* > */dev/sd**X**

This command will write ISO file directly to the USB. 

> To restore the USB drive as an empty, usable storage device after using the Arch ISO image, the ISO 9660 filesystem signature needs to be removed by running ```wipefs --all /dev/sdx``` as root, before repartitioning and reformatting the USB drive. _(from Arch wiki)_

## Booting from USB

Plug your USB drive in and reboot your computer. You may need to update your boot order in order to boot from USB. To do so, smash F2, F10, F11, F12, F1, F3, F8, F7, F4, F5, F6 or F9 key repeatedly until you see the UEFI shell, then update the boot order.

When your computer boots from Arch Live CD, select the first entry in the list.

## First steps

### 1. First and foremost load the correct keymap using ```loadkeys``` command.
List all available keymaps using:

```sh
# ls /usr/share/kbd/keymaps\**\*.map.gz
# loadkeys trq
```

### 2. Check boot mode
```sh
# ls /sys/firmware/efi/efivars
```

If the directory is listed without errors, then the system's booted in UEFI mode. If directory doesn't exist, system may be in BIOS mode.

### 3. Connect to the Internet

#### WIFI

```sh
# rfkill unblock all
# iwctl station wlan0
# iwctl station wlan0 scan
# iwctl station wlan0 get-networks
# iwctl station wlan0 connect MyNetworkSSID
```

If ethernet cable is plugged in, connection should be activated automatically.

Check the connection:
```sh
# ping -c 3 archlinux.org
```

### 4. Set timezone and sync time using NTP 

> \# timedatectl set-timezone *Region/Subregion*
> \# timedatectl set-ntp true

```sh
# timedatectl set-timezone Turkey
# timedatectl set-ntp true
```

Use ```timedatectl list-timezones``` to list all timezones.
Check status using ```timedatectl status```.

## Creating partitions

First of all list available block devices and their partitions using ```lsblk``` command

My current setup is partitioned like this:

| NAME      | MOUNTPOINTS | DESCRIPTION        |
|-----------|-------------|--------------------|
| nvme0n1p1 | /efi        | EFI Partition      |
| nvme0n1p2 |             | Microsoft reserved |
| nvme0n1p3 |             | Windows            |
| nvme0n1p4 |             | Backup             |
| nvme0n1p5 |             | Windows recovery   |
| nvme0n1p6 | swap        | Linux swap         |
| nvme0n1p7 | /           | Linux filesystem   |

You strictly need to create a Linux filesystem to mount the root partition, but you can split different mountpoints (/home, /tmp... etc) to different partitions. A swap partition might not be needed, but many posts and discussions I've read convinced me to set one up.

Use `cfdisk` (a TUI alternative to fdisk) to edit the partition table.   

**DO NOT delete or format the ESP!** ESP contains bootloaders of other installed OSes. Altering this partition will result in data loss.

Assuming that you've created the partitions, double checked the sizes and made sure that everything is correctly implemented, we can now format the partitions.

> \# mkfs.ext4 */dev/partition_where_Linux_will_reside*

> \# mkswap */dev/swap_partition*

```sh
# mkfs.ext4 /dev/nvme0n1p7
# mkswap /dev/nvme0n1p6
```

## Installing base system

Now it's time to ***really*** install the system. Begin by mounting freshly created & formatted partitions:

> \# mount  */dev/Linux_partition* /mnt

You may want to change ESP mountpoint, but I prefer /efi for its simplicity. Please check [this Arch wiki article](https://wiki.archlinux.org/title/EFI_system_partition) and [this Reddit post](https://www.reddit.com/r/archlinux/comments/t6eoyu/where_do_you_mount_your_efi_partition_and_why/)

> \# mount  */dev/EFI_partition*   /mnt/efi

> \# mount  */dev/home_partition*  /mnt/home **(an example)**

> \# swapon */dev/swap_partition*

```sh
# mount  /dev/nvme0n1p1 /mnt/efi
# mount  /dev/nvme0n1p7 /mnt
# swapon /dev/nvme0n1p6
```

Use ```pacstrap``` to install base system to /mnt (where Linux partition is mounted):

> \# pacstrap /mnt base linux linux-firmware... ***(check Packages.md to view every program)***

Now create the filesystem table (fstab) by calling:

> \# genfstab -U /mnt >> /mnt/etc/fstab

And check the output for errors:

> \# cat /mnt/etc/fstab

## Chrooting into system

Change root into the new system using ```arch-chroot /mnt``` and do the initial configuration.

### 1. Set the timezone

While in chroot, timedatectl won't work. To set the timezone, you should manually create a symbolic link to /etc/localtime.
Using the timezone from earlier,

> \# ln -sf /usr/share/zoneinfo/*Region*/*City* /etc/localtime

> \# hwclock --systohc

```sh
# ln -sf /usr/share/zoneinfo/Turkey /etc/localtime
# hwclock --systohc
```

### 2. Set localization

1. Uncomment the locales you want to use in ```/etc/locale.gen```, then run ```locale-gen``` to generate those locales. 

> \# vim /etc/locale.gen

2. Create the ```/etc/locale.conf``` and set the variables according to your needs. 

> LANG=*locale*

> LANGUAGE=*same_locale*

```
LANG=en_GB.UTF8
LANGUAGE=en_GB.UTF8
```

3. If you've changed loaded keymap at the beginning, you need to make this change persistent. To do so, create the ```/etc/vconsole.conf``` and append 

> \# vim /etc/vconsole.conf

> KEYMAP=*keymap*

You can also permanently set virtual console (tty1 -> tty6) font using this file. To do so append:

> FONT=*consolefont*

 - To list available console fonts use: ```ls /usr/share/kbd/consolefonts/*```
 - To list available keymaps use: ```ls /usr/share/kbd/keymaps/**/*```

```
KEYMAP=trq
FONT=eurlatgr.psfu.gz
FONT_MAP=8859-9
```

### 3. Network configuration

Set your hostname:

> \# echo *hostname* >> /etc/hostname

Create and edit the hosts file:

> \# touch /etc/hosts

> \# vim   /etc/hosts

Append these lines:
> 127.0.0.1        localhost

> ::1              localhost

> 127.0.0.1        *your_hostname*

### 4. Add password for root user

> \# passwd

Using root account for long periods is not advised. You should add a normal user after booting into the system.

## 5. Installing the GRUB bootloader

Before starting grab required packages:
> \# pacman -S grub efibootmgr os-prober

Start by installing GRUB in EFI system partition:
> \# grub-install --target=x86_64-efi --efi-directory=*/path/to/esp* --bootloader-id=GRUB

Now that you've GRUB installed, it's time to generate boot entries:

os-prober is disabled by default in ```/etc/default/grub```. You must edit it to find other OSes and add them to GRUB boot list.

> \# vim /etc/default/grub
Find ```GRUB_DISABLE_OS_PROBER=false``` entry and uncomment it. 

Also while you're here, you can change ```GRUB_CMD_LINUX="bla bla bla"``` entry to ```GRUB_CMD_LINUX=""```. This will show kernel messages while booting, I find it useful when my boot goes wrong and I get stuck.

```sh
# grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=GRUB
# vim /etc/default/grub
# grub-mkconfig -o /boot/grub/grub.cfg
```

The first step of installation is done :)

```exit``` chroot, unmount previously mounted devices, then reboot your machine.

## First boot

After the first boot, you'll be greeted by the login program running on bash. Use your root credentials to log in.

```
Login: root
Password:

root@hostname:~$ 
```

You're literally free after this point, you can install any package, customize your desktop and do stuff.

I use [i3 window manager](https://github.com/i3/i3), [polybar](https://github.com/polybar/polybar) and [rofi](https://github.com/davatorium/rofi) in graphical interface. For console I prefer [fish](https://github.com/fish-shell/fish-shell) over bash. To install and manage AUR packages, I use [yay](https://github.com/Jguer/yay).

You can check [Packages.md](https://gist.github.com/fybalaban/d188fe0de48ea92c5b74715471e0f39b#file-packages-md) to see the packages I install.

You can continue to read from Configuration.md where I've configured the programs I've installed and the system itself.

Thanks for reading!

Ferit Yigit BALABAN <[fyb@duck.com](mailto:fyb@duck.com)\>, 2022
