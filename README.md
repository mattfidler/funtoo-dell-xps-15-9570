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


```sh
parted -a optimal /dev/nvme0n1
```

This is the partition that I hope to be making:

| # | Usage | Size |
|---
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
mkpart primary 103 603
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


# References

- https://guyrobottv.wordpress.com/2017/04/18/installing-gentoo-linux-on-zfs-with-nvme-drive-part-1/
