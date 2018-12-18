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
mkdir boot/efi
mount /dev/nvme0n1p2 boot/efi
```

Download stage3 tarball from https://www.funtoo.org/Intel64-skylake



# References

- https://guyrobottv.wordpress.com/2017/04/18/installing-gentoo-linux-on-zfs-with-nvme-drive-part-1/
- https://www.funtoo.org/ZFS_Install_Guide
