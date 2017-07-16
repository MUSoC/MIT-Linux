# HOW TO

This part will explain on how to edit your own LiveCD
NOTW: we will take mit-linux.iso as the name of ISO use. Change it with whatever file name you are using

### Create a folder and mount the iso
For doing so, we will execute the following command
```
mkdir mnt
sudo mount -o loop mit-linux.iso mnt
```

### Copy the contents to another folder
Here, we will copy the contents to another folder named extraxt to use for editing
```
mkdir extract
sudo rsync --exclude=/casper/filesystem.squashfs -a mnt/ extract
```

### Extract and copy the file system
We will need to extract the filesystem to use as environment for editing
```
sudo unsquashfs mnt/casper/filesystem.squashfs
sudo mv squashfs-root edit
```

### Enable internet access
We will need internet connection for various file transfers and installations
for that, we need to copy /etc/resolv.conf and mount /dev
```
sudo cp /etc/resolv.conf edit/etc/
sudo mount --bind /dev/ edit/dev
```

### Create chroot environment
A chroot environment is where all the magic happens. You can boot into filesystem and modify using command line
```
sudo chroot edit
```

### Mount important directories
Without saying, you will need these directories for proper functioning
```
mount -t proc none /proc
mount -t sysfs none /sys
mount -t devpts none /dev/pts
export HOME=/root
export LC_ALL=C
dbus-uuidgen > /var/lib/dbus/machine-id
dpkg-divert --local --rename --add /sbin/initctl
ln -s /bin/true /sbin/initctl

```

### Do the magic
This is where all the magic happens. Install/remove apps, change setting, wallpapers, etc.

### Cleanup
We now need to cleanup the temporary files created by the magic we did
for doing so, following commands will help
```
apt-get autoremove && apt-get autoclean
rm -rf /tmp/* ~/.bash_history
rm /var/lib/dbus/machine-id
rm /sbin/initctl
dpkg-divert --rename --remove /sbin/initctl
```
### Unmount directories
We will have to now unmount the directoried we previously mounted
```
umount /proc || umount -lf /proc
umount /sys
umount /dev/pts
exit
```
NOTE: the exit command exits you from the chroot environment
```
sudo umount edit/dev
```

### Packing the filesystem
```
sudo chmod +w extract/casper/filesystem.manifest
sudo chroot edit dpkg-query -W --showformat='${Package} ${Version}\n' | sudo tee extract/casper/filesystem.manifest
sudo cp extract/casper/filesystem.manifest extract/casper/filesystem.manifest-desktop
sudo sed -i '/ubiquity/d' extract/casper/filesystem.manifest-desktop
sudo sed -i '/casper/d' extract/casper/filesystem.manifest-desktop
sudo mksquashfs edit extract/casper/filesystem.squashfs -b 1048576
printf $(sudo du -sx --block-size=1 edit | cut -f1) | sudo tee extract/casper/filesystem.size
```
NOTE: add  -Xcompression-level # to for mksquashfs compression level. (1-9) 

### Generate md5sum and ISO
```
cd extract
sudo rm md5sum.txt
find -type f -print0 | sudo xargs -0 md5sum | grep -v isolinux/boot.cat | sudo tee md5sum.txt
sudo genisoimage -D -r -V "$IMAGE_NAME" -cache-inodes -J -l -b isolinux/isolinux.bin -c isolinux/boot.cat -no-emul-boot -boot-load-size 4 -boot-info-table -o ../mit-linux-generated.iso .
cd ..
```

### Done
The new generated ISO will be named mit-linux-generated.iso
