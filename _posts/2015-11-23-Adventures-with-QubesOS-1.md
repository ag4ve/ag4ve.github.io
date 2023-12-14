---
pin: true
layout: post
title: "Adventures in QubesOS (part 1)"
date: 2015-11-23
categories:
  - qubes
  - os
tags:
  - security
  - anaconda
  - luks
image:
  path: //d1u9biwaxjngwg.cloudfront.net/highlighted-code-showcase/peak-140.jpg
---

# Qubes Install

I'm mainly writing this so that I don't forget. The installer for Redhat/Fedora/Qubes (Anaconda) doesn't do well when you try to do an unusual partition scheme. How unusual? Lets go with:
* mdadm raid for everything
* luks encrypted root and swap
* provision an efi secure boot partition (Qubes doesn't currently support this)
* use 4x 3TB disks

Shouldn't be that hard, right? Debian's installer handles this pretty well after all. If installing Redhat/Fedora, you need to boot into rescue mode and get to a shell. If installing Qubes, use a Fedora disk to do the above.

So, what we want is:
* A bootloader partition on all disks (installer will complain if you don't have this on a GPT disk - I'm assuming this vs MBR)
* Secure boot partition on all disks (shouldn't change much, so re-syncing data around shouldn't be a big deal)
* Boot partition (raid1)
* LUKS partition (raid5)

Inside LUKS:
* LVM with:
  - swap
  - root

## Lets begin:

```console
 # gdisk /dev/sda
 Go into expert mode (x) and set the start block to 34 and exit expert mode (m)
 Create the boot loader partition: n, start is 34 end is 2047, type is EF02 [1]
 Create the EFI partition: n, default start, give it a hundred megs or so (EFI images can't be >40M - I did ~128M), type is EF00
 Create the raid boot partition: n, default start, and give it 500-1024M, type FD00
 Create user space partition: n, default start, to the end of the disk, type FD00
 (There's a sort (s) option if you did these out of order and want the partition number relative to where it is physically on the disk)
 Write the data: w
```
{: file="gdisk.out" }

Note: this is documented more [1]

## Backup and copy the partition table:

```console
 # sgdisk --backup=table /dev/sda
 # for i in b c d; do sgdisk --load-backup=table /dev/sd$i ; done
```
{: file="part-copy.out" }

I then went back in to gdisk for each of the disks and changed the EFI partition's name (#2 for me) to "ESB <disk number>" as I figured when using efibootmgr's -L, a common name might confuse things (or just error). Maybe I'm wrong - I don't know yet. [2]

Note: I don't believe that creating the filesystem is required here as the installer wants to do that part for you, but I like to do it anyway.

## Create the EFI filesystem:

>  # for i in a b c d; do mkfs.vfat "/dev/sd${i}2" ; done

## Create the /boot raid1 and filesystem [2]:

>  # mdadm --create /dev/md0 --level=1 --raid-devices=4 --metadata=0.9 /dev/sd[abcd]3

Notice the number for "raid-devices" and "abcd" if you have more or fewer disks. This shouldn't take very long as long as you were sane with the space given to /boot.

## Create /boot filesystem

>  # mkfs.ext3 /dev/md0

## Create the raid for user space [2]:

>  # mdadm --create /dev/md1 --level=5 --raid-devices=4 /dev/sd[abcd]4

And go to sleep, go to work, go for a *very* long dinner, watch a Star Wars marathon: this took >5 hours on my computer. If you're curious (keep in mind: your screen will go to sleep and setterm didn't seem to be on the rescue disk):

## Progress

>  # watch cat /proc/mdstat

After it is finished building your second raid device, think up a nice long password (that you won't forget) and ...

## Create and open a LUKS system (no reference):

```console
 # cryptsetup luksFormat /dev/md1
 # cryptsetup luksOpen /dev/md1 main
```
{: file="luks-format.out" }

## Create LVM on top of that [2]:

```console
 # pvcreate /dev/mapper/main
 # vgcreate Main /dev/mapper/main
 # lvcreate -L 32G Main -n swap
 # lvcreat -L 20G Main -n dom0
```
{: file="lvm-create.out" }

Note: sizes may vary - this is what I did for Qubes (not sure I needed that much swap though). So if this is for a larger OS, 20G might not be what you want.

## Create filesystems:

```console
 # mkfs.ext4 /dev/mapper/dom0
 # mkswap /dev/mapper/swap
```
{: file="mke2fs.out" }

## Close everything up:

```console
 # vgchange -a n Main
 # cryptsetup luksClose main
 # mdadm --stop /dev/md1
 # mdadm --stop /dev/md0
```
{: file="cleanup.out" }

Reboot into the Qubes (or Redhat/Fedora) installer. Assign "0" to /boot and ESB 1 to /boot/efi. Unlock LUKS, you'll see  your two LVM volumes - assign swap to be swap space and dom0 to be /. And install.

## Links

0. https://www.thomas-krenn.com/en/wiki/Mdadm_recovery_and_resync

References:
1. https://wiki.archlinux.org/index.php/GRUB#GUID_Partition_Table_.28GPT.29_specific_instructions
2. http://askubuntu.com/questions/660023/how-to-install-ubuntu-14-04-64-bit-with-a-dual-boot-raid-1-partition-on-an-uefi/660038


[1]: https://wiki.archlinux.org/index.php/GRUB#GUID_Partition_Table_.28GPT.29_specific_instructions
[2]: http://askubuntu.com/questions/660023/how-to-install-ubuntu-14-04-64-bit-with-a-dual-boot-raid-1-partition-on-an-uefi/660038

