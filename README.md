Read-only Root-FS with overlayfs for Raspian / Raspberry Pi OS
==============================================================
This repository contains some useful files that allow you to use a Raspberry PI using a readonly filesystem.

This files contains some ideas and code of the following projects:
- https://gist.github.com/niun/34c945d70753fc9e2cc7
- https://github.com/chesty/overlayroot

Congratulate the original authors if these files works as expected. Too, you can congratulate to me to join all files in a one repository and do some changes allowing to use the root and boot filesytem in readonly mode.

Setup
=====
To use this code, you can execute the follow commands:

```
sudo su
cd ~

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
rsync -va root-ro/rootfs/ /
mkinitramfs -o /boot/initrd.gz
echo -e "\n# root-ro\ninitramfs initrd.gz" >> /boot/config.txt

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

If you plan on doing greater changes (like `apt install`), it's probably the best choice to reboot the Pi with read-write enabled, do your changes and boot back into read-only again. Basically all you have to do is to add a comment mark to the "initramfs initrd.gz" line to the /boot/config.txt file. There are convenience scripts that will do this for you: 

- `sudo reboot-ro`
- `sudo reboot-rw`

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

Use with Docker
===============

Docker uses overlay2 internally, this is not compatible with root-ro in default configuration. The easiest to fix this is probably using a different storage-driver such as _fuse-overlayfs_. This might have [significant performance issues][fuse-overlayfs-performance] though:  

```
sudo reboot-rw

apt update
apt install fuse-overlayfs
```

Edit `/etc/docker/daemon.json`:

```json
{
  "storage-driver": "fuse-overlayfs"
}
```

```
systemctl restart docker
reboot-ro
```

Test with:

```
reboot-rw
docker run -dt --name echo --restart always alpine sh -c 'while sleep 10; do date; done'
docker logs --tail 100 -f echo
reboot-ro
docker logs --tail 100 -f echo
reboot
docker logs --tail 100 -f echo
```

If setup correctly you should see some lines that were added when the container was in rw mode, and some new lines that were added in ro mode. The ro lines will be reset by every reboot.

[overlay2-error]: https://stackoverflow.com/a/67954081/1997890
[docker-paths]: https://www.freecodecamp.org/news/where-are-docker-images-stored-docker-container-paths-explained/
[docs-overlayfs-driver]: https://docs.docker.com/storage/storagedriver/overlayfs-driver/
[fuse-overlayfs-performance]: https://news.ycombinator.com/item?id=26102070