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
|---|-------|------|
| 1 | Bios Partition | 3 MiB |
| 2 | EFI Partition | 100 MiB |
| 3 | Boot Partition | 500 MiB |
| 4 | Swap partition | 40,000 MiB |
| 5 | Root Partition | All Remaining |

Once parted starts these are the commands that you can type

```
unit mib
mklabel gpt
mkpart primary 1 3
mkpart primary 3 103
mkpart primary 103 603
mkpart primary 603 40603
mkpart primary 40603 -1
name 1 grub
name 2 esp
name 3 boot
name 4 swap
name 5 rpool
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
zpool create -f -d -o feature@async_destroy=enabled -o feature@empty_bpobj=enabled -o feature@lz4_compress=enabled -o feature@multi_vdev_crash_dump=enabled -o feature@spacemap_histogram=enabled -o feature@enabled_txg=enabled -o feature@hole_birth=enabled -o feature@extensible_dataset=enabled -o feature@embedded_data=enabled -o feature@bookmarks=enabled -o feature@filesystem_limits=enabled -o feature@large_blocks=enabled -o feature@sha512=enabled -o feature@skein=enabled -o feature@edonr=enabled -o feature@userobj_accounting=enabled -o ashift=12 -o cachefile=/tmp/zpool.cache -O compression=lz4 -O normalization=formD  -O atime=off -m none -R /mnt/funtoo rpool /dev/nvme0n1p5

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

zpool create -f -d -o ashift=12 -o cachefile=none -m /boot -R /mnt/funtoo boot /dev/nvme0n1p3

# Confirm the list
zfs list -t all

```

Create and enable the swap

```sh
mkswap /dev/nvme0n1p4
swapon /dev/nvme0n1p4
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
tar -xpvf ~/Downloads/stage3-*.tar.xz
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

This will setup the `/boot`, `/boot/efi`, swap, and `/tmp` directory in the `/etc/fstab` file

```sh
echo /dev/nvme0n1p2    /boot/efi       vfat            defaults,noauto        1 2 > /etc/fstab
echo /dev/nvme0n1p4    none            swap            sw                     0 0 >> /etc/fstab
echo tmpfs   /tmp         tmpfs   nodev,nosuid,size=2G          0  0 >> etc/fstab
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

```

## Add networking support

```sh
## The Desktop flavor is needed for networkmanager to work.
epro flavor desktop
emerge -auND @world ## Accept the configuration change
etc-update ## Use -3 to merge the updates
emerge -auND @world ## install the packages
emerge linux-firmware
emerge -a networkmanager # May break gnome3 need to uninstall consolekit later...?
etc-update # -3 to accept changes
emerge networkmanager
## See wiki.gnome.org/Projects/ConsoleKit
rc-update add NetworkManager default
```

## Making the terminal readable

```sh
emerge media-fonts/terminus-font
```

Then edit `/etc/conf.d/consolefont` to have:

```
consolefont="ter-132b"
```

After execute the command
```sh
# Enable legable fonts as soon as possible
rc-update add consolefont sysinit
```

## Grub and bootloader


```sh
#Create a ZFS-friendly initramfs

emerge --oneshot sys-kernel/genkernel
rm -rf /boot/initramfs* # Remove Debian's default initramfs
genkernel initramfs --no-clean --no-mountboot --makeopts=-j12 --kernel-config=/usr/src/linux/.config --zfs

Confirm the presence of the new initramfs:
ls /boot/*genkernel*
```

This works with one kernel version.

Now add grub with zfs support

```sh
##GRUB 2 must be built with support for ZFS Storage Pools on a 
## single disk. This is achieved using the 'libzfs' USE flag.

echo "sys-boot/grub libzfs" >> /etc/portage/package.use
## Add `GRUB_PLATFORMS` to the `make.conf` if not present
echo 'GRUB_PLATFORMS="efi-64 pc"' >> /etc/portage/make.conf

emerge grub
touch /etc/mtab
grub-probe / # Make sure this is zfs
```


## Making grub readable

```
emerge media-fonts/dejavu # may need to re-emerege after X installs

mkdir -p /boot/fonts

grub-mkfont --output=/boot/fonts/DejaVuSansMono48.pf2 \
  --size=48 /usr/share/fonts/dejavu/DejaVuSansMono.ttf
```

Then edit `etc/default/grub` and add the line:

```
GRUB_FONT=/boot/fonts/DejaVuSansMono48.pf2
```

Make sure that the config file has:

```
GRUB_CMDLINE_LINUX="dozfs real_root=ZFS=rpool/ROOT/funtoo"
```


After that update the grub configuration:

```sh
emerge sys-apps/gptfdisk
sgdisk -g -a1 -n2:48:2047 -t2:EF02 -c2:"BIOS boot partition" /dev/nvme0n1p1
partx -u /dev/nvme0n1 ## Refresh the partitions
grub-install --efi-directory=/boot/efi /dev/nvme0n1
grub-mkconfig -o /boot/grub/grub.cfg
```

You need to make sure it picks up the generated `initramfs` otherwise the system will not boot.

## Making colemak the default console keyboard
edit `/etc/conf.d/keymaps` to have the following line

```
keymap="en-latin9"
```
This way I can type without having to look at my keyboard.

## Time zone configuration

You should link to the local time-zone.  For me it is US Central Time with daylight savings:

```sh
ln -sf /usr/share/zoneinfo/CST6SDT /etc/localtime
```

## Locale setup

For me, the US and UTF locales are appropriate so this does not need to change.

## Final Configuration and reboot

```sh
## make sure root has a password
passwd
```

Exit and reboot

```sh
exit
umount -lR {dev,proc,sys}
umount boot/efi
cd /
zpool export boot
zpool export rpool
reboot
```

## Setting up networking and setup reset state.

After rebooting setup your WiFi internt using `nmtui`.  Once you have
set this up, and check it is working using `ping www.google.com`, save
your zfs state in case you need to reset to this point.

```sh
# First import the boot pool
zpool import boot
zpool snapshot boot@install
zpool snapshot rpool/ROOT/funtoo@install
```

## Changing the kernel options so that Xorg will work
Without dropping some of the frame-buffer support Xorg doesn't really
work. Because of this, you will need to compile your own kernel
removing all the framebuffer options except simple frame-buffer.  You
can also build-in the nvme options and change the processor type to an
intel xenon (for possible gains in performance)

To do this, after a boot
```sh
zpool import boot # import the boot pool
mount /boot/efi
```

Then you can modify the kernel options by:

```sh
zpool import boot
emerge genkernel intel-microcode
cd /usr/src/linux
zcat /proc/config.gz > .config
genkernel kernel --no-mrproper --no-clean --menuconfig --no-mountboot --makeopts=-j12 --zfs --real-root=ZFS=rpool/ROOT/funtoo
ego sync
emerge zfs-kmod zfs linux-firmware sys-firmware/intel-microcode # Update zfs modules
genkernel initramfs --no-mrproper --no-clean --no-mountboot --makeopts=-j12 --zfs --real-root=ZFS=root/ROOT/funtoo --firmware
## grub-install --efi-directory=/boot/efi /dev/nvme0n1
grub-mkconfig -o /boot/grub/grub.cfg


## In the future....
# echo "sys-kernel/debian-sources binary" >> /etc/portage/package.use
# emerge debian-sources

## Reboot and cross your fingers... :)
exit
umount -lR {dev,proc,sys}
umount boot/efi
cd /
zpool export boot
zpool export rpool
reboot

```

## Recovery

If for some reason you need to restart using the Ubuntu installer USB,
here are the commands you will need to issue from the ubuntu terminal:

```sh
sudo bash
apt-add-repository universe
apt-get install zfs-initramfs
zpool import rpool -R /mnt/funtoo -o cachefile=/tmp/zpool.cache # needed to rebuild kernel
zpool import boot -R /mnt/funtoo
cd /mnt/funtoo
mount -t proc none proc
mount --rbind /sys sys
mount --rbind /dev dev
mount /dev/nvme0n1p2 boot/efi
cp /etc/resolv.conf /mnt/funtoo/etc/
cp /tmp/zpool.cache /mnt/etc/zfs/ # needed to rebuild kernel
# We are now ready to chroot.

chroot /mnt/funtoo /bin/bash
env-update
source /etc/profile
export PS1="(chroot) $PS1"

## If you need to restore either the root or boot partition you can by

zfs rollback boot@install

zfs rollback rpool/ROOT/funtoo@install

```

# User configuration

Need to add to `plugdev` for wifi access.  Also need `nm-appled` for gtk and `plasma-nm` for KDE


# References

- Gentoo nvme install guide https://guyrobottv.wordpress.com/2017/04/18/installing-gentoo-linux-on-zfs-with-nvme-drive-part-1/
- ZFS install guide https://www.funtoo.org/ZFS_Install_Guide
- Information on Console Fonts https://wiki.archlinux.org/index.php/HiDPI and https://www.funtoo.org/Fonts
- Colemak Support https://blog.jolexa.net/post/gentoo-colemak-keymap-support/
- Wifi networking https://www.funtoo.org/Install/Network
- Grub Fotns http://blog.wxm.be/2014/08/29/increase-font-in-grub-for-high-dpi.html
- Dell XPS 15 9560 Gentoo wiki https://wiki.gentoo.org/wiki/Dell_XPS_15_9560
