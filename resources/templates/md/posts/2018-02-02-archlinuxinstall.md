{:title "Quick Install Arch Linux"
 :layout :post
 :tags  ["Guides/Instructions"]
 :toc true}



## Introduction

[Arch linux](https://www.archlinux.org/) is probably the best Operating System out there for power users, programmers, and people who like to do things the hard way. [Gentoo](https://www.gentoo.org/get-started/about/) is another great DIY distro for those with a few hours - days to spare.  I like to have any given system up and running within an hour or less and Arch Linux, once mastered, shouldnt take anyone too long to install and configure.  Its install process compared to installers from other battle-tested distroburions, like Debian, is a lot more manual and therefore, more customizable.  A Linux install, regardless of the distro, follows the same basic steps.  With Arch you do these yourself and essentially build your own system from the ground up.  

This guide is for those of us who love Arch Linux and want a trimmed installation guide with just the commands needed to get a minimal install on a modern x86_64 EFI system up and running speedily.  The goal of this guide is to be less wordy than most Arch Linux install guides so that one can quickly go from command to command without much searching.  If you have questions consult the official [Arch Linux install guide](https://wiki.archlinux.org/index.php/Installation_guide).





## Writing to a USB

Insert the USB to be used for writing.  Run:

```bash
lsblk
```

to find your USB drive's letter.


Run the command below from a Linux terminal, of course replacign the "X" with the letter of your USB drive

```bash
sudo dd if="put your arch iso here" of=/dev/sdX bs=4M status=progress && sync
```




## Boot up the Installer

You should be greeted by the "root@archiso" prompt.

Run:

```bash
efivar -l
```

If you are presented with an alarmingly large wall of text that means you booted into EFI mode, congrats.

If you arent connected to ethernet, you should connect to ethernet.  

If you are stubborn and want to setup wireless this second refer to [here](https://wiki.archlinux.org/index.php/Wireless_network_configuration)
Then run:

```bash
ping archlinux.org
```

If you receive some responses then you are ready to proceed to the next step.  If not consult [here](https://wiki.archlinux.org/index.php/Network_configuration#Device_driver)




## Keymaps

If you do not use the standard English Qwerty layout run:

```bash
loadkeys <your 2 letter layout here>
```

Else, continue to the next section




## Partitioning

I find parted to be the easiest tool for the job.  I typically stick with the simple boot, root, swap partition architecure.  Your swap space should generally be the same as your RAM if you want to be able to hibernate a computer under a full workload.  

List the current Drives.  You don't want to install Arch onto your USB do you ;)

```bash
lsblk
```

Your harddrive/ssd should be /dev/sda but if not remember its letter.  /dev/sda will be used for the remainder of the guide.

Open parted with reference to your drive:

```bash
parted /dev/sda
```

Run the print command within parted to find your disk's size.

```bash
(parted) print
```

Wipe your entire disk, and create a GPT table.

```bash
(parted) mklabel gpt
```

Create your partition table, replace size with the value of your total disk space - your RAM

```bash
(parted) mkpart ESP fat32 1049kB 538MB
(parted) set 1 boot on

(parted) mkpart primary ext4 538MB [SIZE]
(parted) mkpart primary linux-swap [SIZE] 100%
(parted) quit
```

Run lsblk again to check your results of the above:

```bash
lsblk /dev/sda
```

Now actually create these partitions via bash:

```bash
mkfs.vfat -F32 /dev/sda1
mkfs.ext4 /dev/sda2
mkswap /dev/sda3
swapon /dev/sda3
```




## Install Arch

Get excited its pacstrap time.

Mount the root "/" partition:

```bash
mount /dev/sda2 /mnt
```

Mount the "/boot" partition:

```bash
mkdir -p /mnt/boot
mount /dev/sda1 /mnt/boot
```

Install the base system:

```bash
pacstrap /mnt base base-devel
```

Generate the fstab:

```bash
genfstab -U -p /mnt >> /mnt/etc/fstab
```

Edit the fstab file so that the / partition ends with 2, the boot ends with 1 and swap ends in 0

If you would like the fstab to be optimized for ssd use add discard to the end of the ext4 partition.

Below is my current fstab

```
# Static information about the filesystems.
# See fstab(5) for details.

# <file system> <dir> <type> <options> <dump> <pass>
# /dev/sda2
UUID    /         	ext4      	rw,relatime,data=ordered,discard	0 2

# /dev/sda1
UUID     	/boot     	vfat      	rw,relatime,fmask=0022,dmask=0022,codepage=437,iocharset=iso8859-1,shortname=mixed,utf8,errors=remount-ro	0 1

# /dev/sda3
UUID    none      	swap      	defaults,pri=-2	0 0
```




## Base System Configuration

Enter the new system with arch-chroot:

To prioritize, we need vim before anything else, this step is very important.

```bash
pacman -S gvim
```


```bash
arch-chroot /mnt
```

Open locale.gen:

```bash
vim /etc/locale.gen
```

Uncomment each locale, the en_US.UTF-8 UTF-8 locale is at the top of the file so uncommenting it here is the easiest place to do so for a standard english language system.

Save and exit the file and generate the locales from bash:

```bash
locale-gen
```

Create the locale.conf file with your chosen locale:

```bash
echo LANG=en_US.UTF-8 > /etc/locale.conf
```

The above works only if you selected the english UTF-8 locale.

Go ahead and export it for the current session as well:

```bash
export LANG=en_US.UTF-8
```

Set the timezone with the below command.  If your timezone differs from mine then browse to /usr/share/zoneinfo/ and you will see all of the zones, cd into the zone and then find your subzone.

```bash
ln -s /usr/share/zoneinfo/Zone/SubZone /etc/localtime
```

Now set the hardware clock:

```bash
hwclock --systohc --utc
```

## System Administration

First set an admin password:

```bash
passwd
```

Name your new system with the following:

```bash
echo good-system-name > /etc/hostname
```

Edit the /etc/hosts file to also refer to your new hostname.  Add these lines if they are not there.  If they are then append your chosen name onto the end of the lines.

```
127.0.0.1 localhost.localdomain localhost good-system-name
::1 localhost.localdomain localhost good-system-name
```

Install Network Manager

```bash
pacman -S networkmanager
systemctl enable networkmanager
```

Use systemd-boot to set up your EFI system

```bash
pacman -S dosfstools
bootctl --path=/boot install
vim /boot/loader/entries/arch.conf
```

This all will have installed a bootloader and opened up a new boot loader conffile for systemd to boot the system.  Add the following lines for a nice speedy boot.

```
title Arch Linux
linux /vmlinuz-linux
initrd /initramfs-linux.img
options root=/dev/sda2 rw elevator=deadline quiet splash resume=/dev/sda3 nmi_watchdog=0
```

Save and exit and now configure systemd boot to find our config file:

```bash
echo "default arch" > /boot/loader/loader.conf
```

Reboot with the following commands:

```bash
exit
reboot
```




## Finishing Up

Create a new user

```bash
groupadd users
useradd -m -g users -G wheel -s /bin/bash goodUsername
```

Install sudo and open up the sudoers file.

```bash
pacman -S sudo
vim /etc/sudoers
```

Uncomment the line:

```
%wheel ALL=(ALL) ALL
```

Save and exit.  This line allows your user to execute commands from roor with: 

```bash
sudo <command>
```

Create a password for your new user with:

```bash
passwd goodUsername
```

Lock up root logins with:

```bash
passwd -l root
```

Update your system and install packer (the best aur helper):

```bash
sudo pacman -Syu
sudo pacman -S fakeroot jshon expac git wget
mkdir packer && cd packer
wget https://aur.archlinux.org/cgit/aur.git/plain/PKGBUILD?h=packer
mv PKGBUILD\?h\=packer PKGBUILD
makepkg
sudo pacman -U packer-*.pkg.tar.xz
cd .. && rm -rf packer
```

You now have a very minimal but complete Arch Linux install.  Look into some [window managers](https://wiki.archlinux.org/index.php/window_manager) or [desktop environments](https://wiki.archlinux.org/index.php/desktop_environment) to install onto your base system to use it as a desktop, or look into using it as a [server](https://wiki.archlinux.org/index.php/Server).





