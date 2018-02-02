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

```bash
