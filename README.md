# funtoo-dell-xps-15-9570

## Using ubuntu recovery

1. Use Region to change keyboard to colemak
2. Connect to Wireless network

## Setting up zfs tools

Setup zfs tools in ubuntu

```sh
sudo bash
apt-add-repository universe
apt-get install --yes zfs-initramfs
```

## Setup Hard Disk

This erases the whole hard disk.  If you need the windows keys to run
Windows in a virutalbox save them before you format the computer.

See [Microsoft's method for keeping the key for a virutal machine](https://answers.microsoft.com/en-us/windows/forum/windows_10-windows_install-winpc/convert-windows-10-activation-to-virtual-machine/284089e3-af42-4b42-acb6-1e63d50a4212)

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
zpool create -f -d -o feature@async_destroy=enabled -o feature@empty_bpobj=enabled -o feature@lz4_compress=enabled -o feature@multi_vdev_crash_dump=enabled -o feature@spacemap_histogram=enabled -o feature@enabled_txg=enabled -o feature@hole_birth=enabled -o feature@extensible_dataset=enabled -o feature@embedded_data=enabled -o feature@bookmarks=enabled -o feature@filesystem_limits=enabled -o feature@large_blocks=enabled -o feature@sha512=enabled -o feature@skein=enabled -o feature@edonr=enabled -o feature@userobj_accounting=enabled -o ashift=12 -o cachefile=/tmp/zpool.cache -O compression=lz4 -O normalization=formD  -O atime=off -m none -R /mnt/gentoo rpool /dev/nvme0n1p5

# Root pool
zfs create -o mountpoint=none -o canmount=off rpool/ROOT
zfs create -o mountpoint=/ rpool/ROOT/gentoo

# Pool for builds
zfs create -o mountpoint=none -o canmount=off rpool/GENTOO
zfs create -o mountpoint=/var/tmp/portage -o compression=lz4 -o sync=disabled rpool/GENTOO/build

# Home pool

zfs create -o mountpoint=/home rpool/HOME


 # Make the root system bootable
 
zpool set bootfs=rpool/ROOT/gentoo rpool

zpool create -f -d -o ashift=12 -o cachefile=none -m /boot -R /mnt/gentoo boot /dev/nvme0n1p3

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
cd /mnt/gentoo
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
cd /mnt/gentoo
mount -t proc none proc
mount --rbind /sys sys
mount --rbind /dev dev

mkdir -p /mnt/gentoo/etc/zfs
cp /tmp/zpool.cache /mnt/gentoo/etc/zfs/zpool.cache
# Make sure that the zpoolcache exists!

# You will also want to copy over resolv.conf in order 
# to have proper resolution of Internet hostnames from 
# inside the chroot:

cp /etc/resolv.conf /mnt/gentoo/etc/
# We are now ready to chroot.

chroot /mnt/gentoo /bin/bash
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
emerge-webrsync
emerge --sync
eselect profile list # I chose gnome
eselect profile set 2
env-update
source /etc/profile
emerge --ask --verbose --update --deep --newuse @world

## Setup locale
ls /usr/share/zoneinfo
## For me (currently)
echo "US/Central" > etc/timezone
emerge --config sys-libs/timezone-data
nano -w /etc/locale.gen # for me I enable all the US English locales
locale-gen
eselect locale list
eselect locale set 6 # for me this is US EN utf
```

## Kernel Configuration

```sh
emerge sys-kernel/gentoo-sources
cd /usr/src/linux
wget https://raw.githubusercontent.com/mattfidler/funtoo-dell-xps-15-9570/master/config .config
make menuconfig
make
make modules
make modules_install
make install
```

## Add ZFS tools, bootloader and grub2

```sh
echo "=sys-fs/zfs-kmod-9999 **" >>/etc/portage/package.accept_keywords
echo "=sys-fs/zfs-9999 **" >>/etc/portage/package.accept_keywords
emerge --ask sys-fs/zfs
# Once it has successfully merged, add the following services to the boot runlevel of OpenRC:
rc-update add zfs-import boot
rc-update add zfs-mount boot
#Add another two services to the default runlevel:

 rc-update add zfs-share default
 rc-update add zfs-zed default

```

Since funtoo abandonded zfs-kmod-9999 is not found or zfs-9999 is not found, you can still dwonload the overlays and use funtoo by using `/var/git/meta-repo/kits/core-kit/sys-fs/zfs-kmod`

At the same time, funtoo's ebuilds are a bit old for what I do, so I went to gentoo.  They should be found in gentoo.

## Add networking support

```sh
## The Desktop flavor is needed for networkmanager to work.
#epro flavor desktop # This only works with funtoo, since I use gentoo, this is no longer needed.
emerge linux-firmware
emerge networkmanager
## See wiki.gnome.org/Projects/ConsoleKit
rc-update add elogind boot
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

emerge --ask sys-kernel/genkernel-next # Make sure plymoth is installed in the use flags
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
zfs snapshot boot@install
zfs snapshot rpool/ROOT/funtoo@install
```

## Changing the kernel options so that Xorg will work
Without dropping some of the frame-buffer support Xorg doesn't really
work. Because of this, you will need to compile your own kernel
removing all the framebuffer options except simple frame-buffer.  You
can also build-in the nvme options and change the processor type to an
intel xenon (for possible gains in performance)

You may have to do this multiple times so I created a script
`/root/kbuild.sh` that does the following:

- Gets the running Kenrel's configuration
- Uses genkernel to generate the new kernel including a call to
  `menuconfig` to modify any options
- Builds the kernel and the initramfs
- Installs the kernel

For the bumblebee support you will need to disable the noveau  drivers and all the framebuffer drivers except the simple frame-buffer.

First, I'm upgrading to the gentoo kernel so:

```
zpool import boot
emerge gentoo-sources
cd /usr/src
rm linux
ln -sf linux-4.19.1-gentoo linux
cd /usr/src/linux
zcat /proc/config.gz > /root/config 
genkernel kernel --no-clean --no-mountboot --menuconfig --makeopts=-j12 --kernel-config=/root/config --install --zfs
emerge --ask @module-rebuild
genkernel initramfs --no-clean --no-mountboot --makeopts=-j12 --kernel-config=/root/config --install --zfs --ramdisk-modules
grub-mkconfig -o /boot/grub/grub.cfg
```

After rebooting, you may want to update the snapshots, and then start
the Xorg setup process.

## Add the correct mix-ins

```sh
echo 'VIDEO_CARDS="intel i965 nvidia"' >> /etc/portage/make.conf
echo 'INPUT_DEVICES="evdev wacom keyboard mouse libinput synaptics"' >> /etc/portage/make.conf
echo 'x11-drivers/nvidia-drivers X acpi compat driver gtk3 tools -kms -pax_kernel -static-libs -uvm -wayland' >> /etc/portage/package.use
echo 'x11-drivers/xf86-video-intel dri dri3 sna udev xvmc -debug -tools -uxa' >> /etc/portage/package.use
echo 'x11-drivers/xf86-video-intel bbswitch video_cards_nvidia -video_cards_nouveau' >> /etc/portage/package.use


I will be using +kde and +gnome flavors for
the desktop.

## Emacs
```sh
echo "app-editors/emacs xft" >> /etc/portage/package.use
emerge -a emacs
emacs --font 'DejaVu Sans Mono-18'
echo "Emacs.font: DejaVu Sans Mono-18" >> ~/.Xresources
xrdb -merge ~/.Xresources
```
## Adding the video to the bootup.

Follow the directions for enabling bumblebee or nvidia.  I have had the most luck with nvidia.  Once you installed nvidia, you will need to add `nvidia` to the boot parameters.  If you don't, you will not be able to see the boot process until nvidia is loaded.

I also install plymouth.  My final grub configuration  at `/etc/default/grub` line `GRUB_CMDLINE_LINUX` shows the early loading of `GRUB_CMDLINE_LINUX`:

```
GRUB_CMDLINE_LINUX="dozfs real_root=ZFS=rpool/ROOT/funtoo nvidia quiet splash acpi_rev_override=1 acpi_osi=Linux pcie_aspm=force drm.vblankoffdelay=1 sci_mod.use_blk_mq=1 mem_sleep_default=deep"
```

Also the recommended `/etc/modprobe.d/i915.conf`

```
options i915 enable_fbc=1 enable_guc=-1 disable_power_well=0 fastboot=1
```

## Adding gestures.

You may emerge `libinput-gestures` to allow configuration of tablet-like gestures for the touch screen under linux.  After follow the guide here:

https://github.com/bulletmark/libinput-gestures

## Using gnome gdm and changing the keyboard layout.
This is not well documented when using wayland;  You log into gdm as root, and change the default keyboard to include colemak and remove the "US" layout.  Then add back the "US" layout.  There will be keyboard selector when you login to gnm after that point.


## Recovery

If for some reason you need to restart using the Ubuntu installer USB,
here are the commands you will need to issue from the ubuntu terminal:

```sh
sudo bash
apt-add-repository universe
apt-get install --yes zfs-initramfs


zpool import -f rpool -R /mnt/funtoo -o cachefile=/tmp/zpool.cache # needed to rebuild kernel
zpool set bootfs=rpool/ROOT/funtoo rpool
zpool import -f boot -R /mnt/funtoo

cd /mnt/funtoo
mount -t proc none proc
mount --rbind /sys sys
mount --rbind /dev dev

mount /dev/nvme0n1p2 boot/efi
cp /etc/resolv.conf /mnt/funtoo/etc/
cp /tmp/zpool.cache /mnt/funtoo/etc/zfs/ # needed to rebuild kernel
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

Need to add to `plugdev` for wifi access, `video` for nvidia access.  Also need `nm-appled` for gtk and `plasma-nm` for KDE


# References

- Gentoo nvme install guide https://guyrobottv.wordpress.com/2017/04/18/installing-gentoo-linux-on-zfs-with-nvme-drive-part-1/
- ZFS install guide https://www.funtoo.org/ZFS_Install_Guide
- Information on Console Fonts https://wiki.archlinux.org/index.php/HiDPI and https://www.funtoo.org/Fonts
- Colemak Support https://blog.jolexa.net/post/gentoo-colemak-keymap-support/
- Wifi networking https://www.funtoo.org/Install/Network
- Grub Fotns http://blog.wxm.be/2014/08/29/increase-font-in-grub-for-high-dpi.html
- Dell XPS 15 9560 Gentoo wiki https://wiki.gentoo.org/wiki/Dell_XPS_15_9560
