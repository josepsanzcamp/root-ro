Read-only Root-FS with overlayfs for Raspian
============================================
This repository contains some useful files that allow you to use a Raspberry PI using a readonly filesystem.

This files contains some ideas and code of the following projects:
- https://gist.github.com/niun/34c945d70753fc9e2cc7
- https://github.com/chesty/overlayroot

Congratulate the original authors if these files works as expected. Too, you can congratulate to me to join all files in a one repository and do some changes allowing to use the root and boot filesytem in readonly mode.

Setup
=====
To use this code, you can execute the follow commands:

```
cd /home/pi
sudo bash

echo Installing all dependencies
apt-get install git rsync gawk busybox bindfs

echo Disabling swap
dphys-swapfile swapoff
dphys-swapfile uninstall
update-rc.d dphys-swapfile disable
systemctl disable dphys-swapfile

echo Cloning repository
git clone https://github.com/josepsanzcamp/root-ro.git

echo Doing the setup
rsync -va root-ro/etc/initramfs-tools/* /etc/initramfs-tools/
mkinitramfs -o /boot/initrd.gz
echo initramfs initrd.gz >> /boot/config.txt

echo Restarting RPI
reboot
```

Save changes to SD card
=======================

## For single files

The simplest way to save single files or folders is to write them directly to the lower directory of the overlayfs. However this doesn't work when your changes to `/` come from something like `apt install`.

Write access to the lower directories of OverlayFS can be enabled using following commands:
```
# /
sudo mount -o remount,rw /mnt/root-ro
# /boot
sudo mount -o remount,rw /mnt/boot-ro
```

Make it read-only again:
```
# /
sudo mount -o remount,ro /mnt/root-ro
# /boot
sudo mount -o remount,ro /mnt/boot-ro
```

## Reboot the Pi with read-write enabled

If you plan on doing greater changes (like `apt install`), it's probably the best choice to reboot the Pi with read-write enabled, do your changes and boot back into read-only again. Basically all you have to do is to add a comment mark to the "initramfs initrd.gz" line to the /boot/config.txt file.

Reboot from read-only to read-write:
```
sudo mount -o remount,rw /mnt/boot-ro
sed -i "s/^\(i\)\(nitramfs[[:blank:]]*initrd.gz\)[[:blank:]]*$/#\1\2/" /mnt/boot-ro/config.txt
reboot
```

Reboot back into read-only:
```
sed -i "s/^#[[:blank:]]*\(initramfs[[:blank:]]*initrd.gz\)[[:blank:]]*$/\1/" /boot/config.txt
reboot
```

## Sync changes from upper into lower directory

This is probably not the best choice and I can't anticipate how well this works with complex changes that involve symlinks or even hardlinks. The basic idea is to 'just copy over' all changed files from the upper directory into the lower directory (i.e. your SD card):

```
sudo mount -o remount,rw /mnt/root-ro
rsync -a --remove-source-files --exclude='/tmp/**' /mnt/root-rw/upper/* /mnt/root-ro/.
sudo mount -o remount,ro /mnt/root-ro
```

Original state
==============
To return to the original state to allow easy apt-get update/upgrade and rpi-update, you need to add a comment mark to the "initramfs initrd.gz" line to the /boot/config.txt file.