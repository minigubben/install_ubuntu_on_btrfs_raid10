This is more or less a curated version of bash history after installing ubuntu on a btrfs raid10
Inspired by https://www.youtube.com/@FIXAPC video on the topic.

# enable ssh
```
sudo su
passwd
vim /etc/ssh/sshd_config
remove # from listening flags and set allow root login to yes 
systemctl restart ssh
```

# partitioning
partition table

```
parted -a optimal /dev/nvme0n1 "mklabel msdos"
parted -a optimal /dev/nvme1n1 "mklabel msdos"
parted -a optimal /dev/nvme2n1 "mklabel msdos"
parted -a optimal /dev/nvme3n1 "mklabel msdos"
```

boot partiton
```
parted -a optimal /dev/nvme0n1 mkpart primary 2 1000
parted -a optimal /dev/nvme1n1 mkpart primary 2 1000
parted -a optimal /dev/nvme2n1 mkpart primary 2 1000
parted -a optimal /dev/nvme3n1 mkpart primary 2 1000
```

root partition
```
parted -a optimal /dev/nvme0n1 "mkpart primary 1000 -1"
parted -a optimal /dev/nvme2n1 "mkpart primary 1000 -1"
parted -a optimal /dev/nvme3n1 "mkpart primary 1000 -1"
parted -a optimal /dev/nvme4n1 "mkpart primary 1000 -1"
```


create btrfs raids
```
mkfs.btrfs -m raid1 -d raid1 /dev/nvme0n1p1 /dev/nvme1n1p1 /dev/nvme2n1p1 /dev/nvme3n1p1 -f
mkfs.btrfs -m raid10 -d raid10 /dev/nvme0n1p1 /dev/nvme1n1p1 /dev/nvme2n1p1 /dev/nvme3n1p1 -f
```

set boot flag
```
parted -a optimal /dev/nvme0n1 set 1 boot on
parted -a optimal /dev/nvme1n1 set 1 boot on
parted -a optimal /dev/nvme2n1 set 1 boot on
parted -a optimal /dev/nvme3n1 set 1 boot on
```

# create btrfs subvolumes
```
mkdir /mnt/btrfsroot
mount /dev/nvme0np2 /mnt/btrfsroot

btrfs subvolume create /mnt/btrfsroot/@
btrfs subvolume create /mnt/btrfsroot/@home
```

# mount btrfs
```
mkdir /mnt/osdrive
mount -o noatime,compress=zstd:1,ssd,subvol=@ /dev/nvme0n1p2 /mnt/osdrive/
```

# Installation
copy read only filesystem
```
cd /rofs/
cp -a -r -f -v * /mnt/osdrive/
```
mount hardware/system files
```
mount --type proc /proc /mnt/osdrive/proc/
mount --rbind /sys /mnt/osdrive/sys
mount --rbind /dev /mnt/osdrive/dev
mount --make-rslave /mnt/osdrive/dev/
mount --make-rslave /mnt/osdrive/sys/
```

chroot to new installation
```
chroot /mnt/osdrive/ /bin/bash
soruce /etc/profiles
```

mount boot and home
```
mount /dev/nvme0n1p1 /boot
mount -o noatime,compress=zstd:1,ssd,subvol=@home /dev/nvme0n1p2 /home/
```

set dns, update packages and install aptitude and a text editor
```
echo "nameserver 8.8.8.8" > /etc/resolv.conf
apt update
apt upgrade
apt install aptitude vim
```

reinstall all packages to ensure correct symlinks
```
aptitude reinstall "~i"
```
# setup fstab
insert uuids to fstab
```
blkid > /etc/fstab
```
edit the fstab file setup the correct mountpoints. For example.
```
# boot
UUID="06e4f8f9-21ca-49d7-bc17-b932cdc675a3" /boot       btrfs   defaults                        0 2

# root and home
UUID="5f96069b-bdb6-41a3-8d41-6e3a47f8ee8e" /           btrfs   subvol=@,compress=zstd:1        0 0
UUID="5f96069b-bdb6-41a3-8d41-6e3a47f8ee8e" /home       btrfs   subvol=@home,compress=zstd:1    0 0
```


# install grub to all drives allow boot from any as they are raided anyway
```
apt install grub2
```
Edit GRUB_CMDLINE_LINUX_DEFAULT in /etc/default/grub to match the following
```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash rootflags=subvol=@ noatime compress=zstd:1 ssd"
```

install grub on each device, this allows us to boot from any device in case of coruption.

```
grub-install /dev/nvme0n1
grub-install /dev/nvme1n1
grub-install /dev/nvme2n1
grub-install /dev/nvme3n1
```

set a root password as we don't have any other users yet.
```
passwd
```

# install kernel
```
apt install linux-image-generic
```


# i ended up in grub cli. To boot i entered
Get uuid of root partition from fstab file
```
cat (hd0,msdos2)/@/etc/fstab
```

```
set root=(hd0,msdos1)
linux /vmlinuz-XX-generic root=UUID=XX rootflags=subvol=@ noatime compress=zstd:1 ssd
initrd /initrd.img-XX-generic
boot
```
initrd and vmlinuz files should be tab-completable

# after booting i had no network, this was becuase no netplan was added.
i added the following file /etc/netplan/00-installer-config.yaml and ran chmod 600 on in.
```
network:
  version: 2
  renderer: networkd
  ethernets:
    enp5s0:
      addresses:
        - 10.20.1.3/24
      routes:
        - to: default
          via: 10.20.1.1
      nameservers:
        addresses:
          - 1.1.1.1
          - 8.8.8.8
```
Once the file is added it can be verified running netplan try and applied running netplan apply
