# funtoo-dell-xps-15-9570

## Using ubuntu recovery

1. Use Region to change keyboard to colemak
2. Connect to Wireless network

## Setting up zfs tools

Setup zfs tools in ubuntu

```sh
sudo bash
apt-add-repository universe
apt-get install zfs-initramfs
```

## Setup Hard Disk

This erases the whole hard disk.  If you need the windows keys to run
Windows in a virutalbox save them before you format the computer.

This setup is very similar to the one Guy Robot talks about and the
ZFS guide on funtoo also talks about.


```sh
parted -a optimal /dev/nvme0n1
```

This is the partition that I hope to be making:

| # | Usage | Size |
|---|
| 1 | Bios Partition | 3 MiB |
| 2 | EFI Partition | 100 MiB |
| 3 | Swap partition | 40,000 MiB |
| 4 | Root Partition | All Remaining |

Once parted starts these are the commands that you can type

```
unit mib
mklabel gpt
mkpart primary 1 3
mkpart primary 3 103
mkpart primary 103 40103
mkpart primary 40103 -1
name 1 grub
name 2 esp
name 3 swap
name 4 rpool
set 1 bios_grub on
set 2 boot on
print
quit
```

Make the EFI partition vfat

```sh
mkfs.fat -F32 /dev/nvme0n1p2
```

Then create the ZFS pools

```
zpool create -f -o ashift=12 -o cachefile=/tmp/zpool.cache -O normalization=formD -O atime=off -m none -R /mnt/funtoo rpool /dev/nvme0n1p3

# Root pool
zfs create -o mountpoint=none -o canmount=off rpool/ROOT
zfs create -o mountpoint=/ rpool/ROOT/funtoo

# Pool for builds
zfs create -o mountpoint=none -o canmount=off rpool/FUNTOO
zfs create -o mountpoint=/var/tmp/portage -o compression=lz4 -o sync=disabled rpool/FUNTOO/build

# Home pool

zfs create -o mountpoint=/home rpool/HOME

# Make the root system bootable

zpool set bootfs=rpool/ROOT/funtoo rpool

# Confirm the list
zfs list -t all

```

Create and enable the swap

```sh
mkswap /dev/nvme0n1p3
swapon /dev/nvme0n1p3
```

Now add `boot/efi`

```sh
cd /mnt/funtoo
mkdir -p boot/efi
mount /dev/nvme0n1p2 boot/efi
```

Download stage3 tarball from https://www.funtoo.org/Intel64-skylake

Then extract the tarball 

```sh
# Typically this is in the ~/Downloads if you download it from firefox, so
cd /mnt/funtoo
tar -xpvf ~/Downloads/stage3-latest.tar.xz
```

Next chroot into funtoo

```sh
cd /mnt/funtoo
mount -t proc none proc
mount --rbind /sys sys
mount --rbind /dev dev

mkdir -p /mnt/funtoo/etc/zfs
cp /tmp/zpool.cache /mnt/funtoo/etc/zfs/zpool.cache
# Make sure that the zpoolcache exists!

# You will also want to copy over resolv.conf in order 
# to have proper resolution of Internet hostnames from 
# inside the chroot:

cp /etc/resolv.conf /mnt/funtoo/etc/
# We are now ready to chroot.

chroot /mnt/funtoo /bin/bash
export PS1="(chroot) $PS1"
```

## Setting up fstab

This will setup the `boot/efi` and swap in the `/etc/fstab` file

```sh
echo /dev/nvme0n1p2    /boot/efi       vfat            defaults,noauto        1 2 >> /etc/fstab
echo /dev/nvme0n1p3    none            swap            sw                     0 0 >> /etc/fstab
 ```

## Update portage

```sh
ego sync
env-update
source /etc/profile
```

## Add ZFS tools, bootloader and grub2

```sh
emerge --ask sys-fs/zfs
# Once it has successfully merged, add the following services to the boot runlevel of OpenRC:
rc-update add zfs-import boot
rc-update add zfs-mount boot
#Add another two services to the default runlevel:

 rc-update add zfs-share default
 rc-update add zfs-zed default
#Create a ZFS-friendly initramfs

emerge --oneshot sys-kernel/genkernel
genkernel initramfs --no-clean --no-mountboot --makeopts=-j12 --kernel-config=/usr/src/linux/.config --zfs
```


# References

- https://guyrobottv.wordpress.com/2017/04/18/installing-gentoo-linux-on-zfs-with-nvme-drive-part-1/
- https://www.funtoo.org/ZFS_Install_Guide
