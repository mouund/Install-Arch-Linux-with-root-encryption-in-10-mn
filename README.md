This guide is not meant to be a reference of any sort. Here are the sources you should be referring to:

* [Installation guide ArchWiki](https://wiki.archlinux.org/title/installation_guide)

* [Encrypt an entire device ArchWiki](https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system)

* [Disk preparation](https://wiki.archlinux.org/title/Dm-crypt/Drive_preparation#Secure_erasure_of_the_hard_disk_drive)

Boot on the USB key inserted with Boot menu (F1,F12,DEL,..) [Which key](https://www.pc83.fr/tools/liste-bios-key-boot-menu-key.html)

As you enter the live environment, I advise you either using bigger font or simply increase the size of current one.

Bigger font:

    # setfont ter-132b

OR Double size:

    # setfont -d 

Set your keyboard layout as needed. eg: For french 

    # loadkeys fr-latin1 

Connect to the internet using [iwctl](https://wiki.archlinux.org/title/Iwd#iwctl)

Find out the HHD, SSD... on which you want to install Arch.

    # lsblk -f

    NAME        MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINTS
    sdx           xx     xx     xx  x xx  
    **nvme0n1**     x:x    0 xG  0 disk  

Our disk will be /dev/nvme0n1. Care, if you are using HDD it can be named /dev/sdx (x is a number depending on how many disks are plugged)

> :warning: All the current data stored on the disk will be wiped.

# Pre encryption

Optional: wipe disk.

Create a container on the disk (block device) you want to use
    # cryptsetup open --type plain -d /dev/urandom /dev/<block-device> to_be_wiped

Verify existance:

    # lsblk
    NAME          MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINT
    nvme0n1   8:0    0  1.8T  0 disk
     └─to_be_wiped 252:0    0  1.8T  0 crypt

Write zeros on the mapped disk container:

    # dd if=/dev/zero of=/dev/mapper/to_be_wiped status=progress bs=1M

# Partitioning

    # fdisk /dev/nvme0n1

First partition boot

    n 
    p # New partition
    ENTER # Default number 1
    ENTER # First sector
    +300M # size of the partition

Second partition root

    n
    p
    ENTER # Default number 2
    ENTER # First sector after parition 1
    ENTER # full disk

Now you should see two partition under your main device when using 

    lsblk -f 

    /dev/nvme0n1
      |_nvme0n1p1 --> the boot one (bootloader etc..)
      |_nvme0n1p2 --> the root one (where all system will sit)

# Formating & Encryption of /root

We assign the good format to each of the two partitions and encrypt the root one:

1 - Root Partition

Encrypt our root partition ( the partition 2)

    # cryptsetup -y -v luksFormat /dev/nvme0n1p2

Open it in a container called root mapped in /dev/mapper/root

    # cryptsetup open /dev/sda2 root

Format it to ext4

    # mkfs.ext4 /dev/mapper/root

Mount the parition to /mnt ( for the installation)

    # mount /dev/mapper/root /mnt

Optional but recommended, test that the decrypting cycle works as intended:

    # umount /mnt
    # cryptsetup close root
    # cryptsetup open /dev/nvme0n1p2 root
    # mount /dev/mapper/root /mnt

2 - Boot partition

We go for an EFI partition.

    # mkfs.fat -F32 /dev/nvme0n1p1

Mount the partition for the intstallation

    # mount --mkdir /dev/nvme0n1p1 /mnt/boot

Note: we mount directly the partition, as there is no encyption on this one.

# Installation

The hard part is behind :D

Install Arch

    # pacstrap -K /mnt base linux linux-firmware

Generate fstab file (mount points)

    # genfstab -U /mnt >> /mnt/etc/fstab

chroot into our new system (use our new install as "/" )

    # arch-chroot /mnt

Timezone & Clock:

    # ln -sf /usr/share/zoneinfo/Region/City /etc/localtime

    # hwclock --systohc

Locales & Keyboard Layout:

> :warning: Edit '/etc/locale.gen' uncomment en_US.UTF-8 UTF-8 if you are French e.g: uncomment fr_FR.UTF-8 UTF-8

    # locale-gen


    # echo LANG=en_US.UTF-8 > /etc/locale.conf 

Assign the keyboard layout you want to use. E.g: french

    # echo KEYMAP=fr-latin1 > /etc/vconsole.conf 

Hostname

    # echo myhostname > /etc/hostname

Initramfs

Generate /etc/mkinitcpio.conf

    # mkinitcpio -P

> :warning: Very important changes in order to decrypt your system at boot, Make the line starting by HOOK looking like this:

    # HOOKS=(base udev autodetect modconf kms keyboard keymap consolefont block encrypt filesystems fsck)

Regen initramfs

    # mkinitcpio -p linux

Set root password:

    # passwd 

Bootloader (GRUB):

Install necessary packages:

    # pacman -S grub efibootmgr
Change the partition in the following line if needed aswell

    # OUR_UUID=$( blkid | grep /dev/nvme0n1p2 | awk '{print $2}')
    # OUR_UUID="${OUR_UUID//\"}"
    # FULL_LINE="GRUB_CMDLINE_LINUX=\"cryptdevice=${OUR_UUID}:root root=/dev/mapper/root\""
    # sed -i 's/GRUB_CMDLINE_LINUX=""//g' /etc/default/grub
    # echo $FULL_LINE >> /etc/default/grub
    # grub-install --target=x86_64-efi --efi-directory=/boot

Exit chroot:

    # exit

Unmount /mnt

    # umount -R /mnt

Reboot

    # reboot

Unplug the USB installation medium before the boot process, you should be greeted by GRUB. Then you will be asked to unlock your root partition. Then you are DONE !
 
