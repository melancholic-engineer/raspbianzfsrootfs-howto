# Raspbian (Buster) ZFS rootfs
A walkthrough for setting up a ZFS (>= 0.8.4) rootfs for Raspberry Pi using Raspbian (Buster)

******

In my exploration on this issue I took inspiration from the following places
* [https://github.com/zfsonlinux/pkg-zfs/wiki/HOWTO-install-Raspbian-to-a-Native-ZFS-Root-Filesystem,-or,-How-I-Learned-to-Love-Data-Integrity][1]
* [https://openzfs.github.io/openzfs-docs/Getting%20Started/Ubuntu/Ubuntu%2020.04%20Root%20on%20ZFS.html][2]
* [https://gist.github.com/mohakshah/b203d33a235307c40065bdc43e287547][3]
* [https://gist.github.com/Alexey-Tsarev/d5809e353e756c5ce2d49363ed63df35][4]

You might yet find them useful.

******

## What you will need
* \>= 4GB sd card
* raspberry pi 
* preferably a USB ssd disk or something similiar. (you can install a zpool on a SD card, but why would you?)
* internet connection
* basic bash console skills

## Where has it been tested on
A RPI4 4GB with Rasbian Buster (4.19.118-v7l+)

## Quick benchmark
RPI4 4GB with Rasbian Buster (4.19.118-v7l+)
Intel 660P 1TB over USB3
Using "[agnostics][9]"

    root@???:/usr/share/agnostics# sh sdtest.sh
    Run 1
    prepare-file;0;0;99902;195
    seq-write;0;0;96376;188
    rand-4k-write;0;0;3987;996
    rand-4k-read;12328;3082;0;0
    Sequential write speed 96376 KB/sec (target 10000) - PASS
    Random write speed 996 IOPS (target 500) - PASS
    Random read speed 3082 IOPS (target 1500) - PASS


## 01. Setting up raspbian on the sd card and boot
You can refer to [raspberrypi.org](https://projects.raspberrypi.org/en/projects/raspberry-pi-setting-up) if needed.

## 02. Uncomment "deb-src" in apt sources.list and update packages
    sudo vi /etc/apt/sources.list
    sudo apt-get update 
    sudo apt-get -y dist-upgrade
    
## 03. Install the following tools and dependencies
    sudo apt-get -y install build-essential fakeroot devscripts autogen libelf-dev debian-keyring busybox initramfs-tools
    sudo apt-get clean
    sudo apt-get -y build-dep zfs-dkms spl-dkms

## 04. Setup a folder for us to work in
    mkdir ~/src
    cd ~/src
    
## 05. Getting sources
I've tried compiling from source for ZFS, and try as I might, I could not get ZFS installed properly on Rasbian.

The make and make install will happen successfully, I could even create zpools and datasets once I loaded the zfs module, but update-initramfs will have problems creating correct initramfs  with zfs properly packaged inside.

Without a working initramfs for ZFS, a ZFS rootfs would not work.

That's when I remember [this][1] , and thought it might be a good idea to create my own packages which will handle the necessary tooling/plumbling needed to get all the initramfs/systemd/etc etc stuff configured properly.

Originally, [this][1] referred to debian repos. I managed to find similiar stuff on [raspbian repos][5]. I thought using the raspbian repos would be a better choice as they will handle the idiosyncracies of raspbian itself better.

This guide focuses on zfs 0.8 and above. I don't think the guide will work below 0.8 due to the need to setup SPL, or something.

If you come by from the future and want to use this guide for more recent versions of zfs, just remember to look for files with the "dsc" extension, i.e., zfs-linux*.dsc

Now, download the needed [dsc file][6].

    dget -x http://raspbian.raspberrypi.org/raspbian/pool/contrib/z/zfs-linux/zfs-linux_0.8.4-1.dsc
    
## 06.1 Get a Good Coffee
The next step will take a while.

## 06.2 Build it.
    cd zfs-linux*
    dpkg-source --commit
    debuild -i -us -uc -b
    cd ..
    
* Note01: The original [guide][1] had an additional "--no-lintian" flag. The debuild I had does not recongnise the flag. I left off the flag with no ill effects. YMMV.
* Note02: debuild might fail and warn you of additional packages to install. Just follow instructions and try again.


## 07. Install Kernel Headers

In case you don't have those:

    sudo apt-get install raspberrypi-kernel-headers

    
## 08.1 Get another Good Coffee
DKMS portions of the next step will take a while, even on RPI4

## 08.2 Install the deb lackages

    sudo dpkg -i libnvpair1linux_0*.deb libuutil1linux_0*.deb libzfs2linux_0*.deb libzfslinux-dev_0*.deb libzpool2linux_0*.deb spl_0*.deb spl-dkms_0*.deb zfs-dkms_0*.deb zfs-initramfs_0*.deb zfsutils-linux_0*.deb zfs-zed_0*.deb

## 09. Reboot
And wait for it to come back.

## 10. Disabling Swap.
There has been reported problems with using swap files on zpools for RPI, so it's better to disable them.
Create a swap partition (not on a zvol!) if you really want swap.

    sudo dphys-swapfile swapoff && \
    sudo dphys-swapfile uninstall && \
    sudo systemctl disable dphys-swapfile

## 11. Partitioning
These instructions will wipe out your disk (you ARE using a separate USB drive right?)
If you have other plans for your disk, just tweak your partitioning accordingly.

Delete Everything

    sudo sgdisk --zap-all $DISK

Create a partition that spans the while disk

    sudo sgdisk     -n1:0:0        -t1:BF00 $DISK
    
More help for sgdisk [here][7]

## 12. Setting up zpool
Based on recomendations from [here][2]

    sudo zpool create -o ashift=12 \
        -O acltype=posixacl -O canmount=off -O compression=lz4 \
        -O dnodesize=auto -O normalization=formD -O relatime=on -O xattr=sa \
        -O mountpoint=/ -R /mnt -f rpool $DISK-part1

I was building the RPI4 system for a battery powered project, and I weighted against adding a additional drive due to space, power, thermal constraints (those SSDs run hot without ventilation. Living in the tropics does not help either.).

I know setting "copies=2" is not a replacement for proper redundancy. I just want to cover my bases for data corruption issues due to power, thermal, electrical issues and outright rough handling.

    sudo zfs set copies=2 rpool

Additional setup

    sudo zfs create -o canmount=off -o mountpoint=none rpool/ROOT
    sudo zfs create -o canmount=noauto -o mountpoint=/ rpool/ROOT/nnkmobile
    sudo zfs mount rpool/ROOT/nnkmobile

At this point, your new rootfs will be hanging off /mnt


## 13. Additional dataset setup
At this point you can actually start copying the existing rootfs over to the new rootfs at /mnt.

If you want to make the system slightly easlier to maintain in the future, you can follow the layout suggestions over [here][2]

If not, you can skip ahead to 14.

These are the datasets I've created.

    sudo zfs create                                 rpool/home
    sudo zfs create -o mountpoint=/root             rpool/home/root
    sudo zfs create -o canmount=off                 rpool/var
    sudo zfs create -o canmount=off                 rpool/var/lib
    sudo zfs create                                 rpool/var/log
    sudo zfs create                                 rpool/var/spool

    sudo zfs create -o com.sun:auto-snapshot=false  rpool/var/cache
    sudo zfs create -o com.sun:auto-snapshot=false  rpool/var/tmp
    sudo chmod 1777 /mnt/var/tmp

    sudo zfs create                                 rpool/opt
    sudo zfs create                                 rpool/srv

    sudo zfs create -o canmount=off                 rpool/usr
    sudo zfs create                                 rpool/usr/local

    sudo zfs create -o com.sun:auto-snapshot=false  rpool/tmp
    sudo chmod 1777 /mnt/tmp

## 14. Modify fstab
Comment out the rootfs "/"

If you used my layout in 13., you will also need to add the following to /etc/fstab:

    rpool/var/log /var/log zfs nodev,relatime 0 0
    rpool/var/spool /var/spool zfs nodev,relatime 0 0
    rpool/var/tmp /var/tmp zfs nodev,relatime 0 0
    rpool/tmp /tmp zfs nodev,relatime 0 0

## 15.1 Prepare more Drinks
Next step will take a while, depending on your SD card.

## 15.2 Rsync your exising rootfs
Thanks to [Arch Linux][8] for the hint.

    sudo rsync -qaHAXS --exclude={"/dev/*","/proc/*","/sys/*","/tmp/*","/run/*","/mnt/*","/media/*","/lost+found"} / /mnt 

## 16. Setting mountpoint order
If you used my layout in 13, you will need to do the following adjustments:

    sudo zfs set mountpoint=legacy rpool/var/spool
    sudo zfs set mountpoint=legacy rpool/var/log
    sudo zfs set mountpoint=legacy rpool/var/tmp
    sudo zfs set mountpoint=legacy rpool/tmp

## 17. Setting up /boot/cmdline.txt
Modify your file to look similiar to this

    dwc_otg.lpm_enable=0 console=serial0,115200 root=ZFS=rpool/ROOT/nnkmobile rootfstype=zfs elevator=noop rootwait rootdelay=5 zfs.zfs_arc_max=67108864

Most important entries are:

    root=ZFS=rpool/ROOT/nnkmobile
    rootfstype=zfs
    rootdelay=5

## 18. Setting up /boot/config.txt
Add the line to /boot/config.txt

  initramfs initrd.img-$KERNEL followkernel
  
You can find out about your kernel using "uname -r"

## 19. Update initramfs
Update your initramfs:

    sudo update-initramfs -c -k $(uname -r)

## 20. Umount everything on /mnt and export zpool
Umount everything on /mnt.

Export your zpool using:

    sudo zpool export -a

## 21. Reboot
Reboot again to boot into the new rootfs

## 22. Celebrate.



    
[1]: https://github.com/zfsonlinux/pkg-zfs/wiki/HOWTO-install-Raspbian-to-a-Native-ZFS-Root-Filesystem,-or,-How-I-Learned-to-Love-Data-Integrity
[2]: https://openzfs.github.io/openzfs-docs/Getting%20Started/Ubuntu/Ubuntu%2020.04%20Root%20on%20ZFS.html
[3]: https://gist.github.com/mohakshah/b203d33a235307c40065bdc43e287547
[4]: https://gist.github.com/Alexey-Tsarev/d5809e353e756c5ce2d49363ed63df35
[5]: http://raspbian.raspberrypi.org/raspbian/pool/contrib/z/zfs-linux/
[6]: http://raspbian.raspberrypi.org/raspbian/pool/contrib/z/zfs-linux/zfs-linux_0.8.4-1.dsc
[7]: https://linux.die.net/man/8/sgdisk
[8]: https://wiki.archlinux.org/index.php/Rsync#File_system_cloning
[9]: https://www.raspberrypi.org/blog/sd-card-speed-test/
