<p align="center">
  <img src="https://img.shields.io/github/license/dutofanim/gentoo_install_guide?style=plastic" />
</p>


# Installation instructions of

```text
     .vir.                                d$b
  .d$$$$$$b.    .cd$$b.     .d$$b.   d$$$$$$$$$$$b  .d$$b.      .d$$b.
  $$$$( )$$$b d$$$()$$$.   d$$$$$$$b Q$$$$$$$P$$$P.$$$$$$$b.  .$$$$$$$b.
  Q$$$$$$$$$$B$$$$$$$$P"  d$$$PQ$$$$b.   $$$$.   .$$$P' `$$$ .$$$P' `$$$
    "$$$$$$$P Q$$$$$$$b  d$$$P   Q$$$$b  $$$$b   $$$$b..d$$$ $$$$b..d$$$
   d$$$$$$P"   "$$$$$$$$ Q$$$     Q$$$$  $$$$$   `Q$$$$$$$P  `Q$$$$$$$P
  $$$$$$$P       `"""""   ""        ""   Q$$$P     "Q$$$P"     "Q$$$P"   Linux
  `Q$$P"                                  """
```

In this script we will cover the installation of [Gentoo Linux](https://www.gentoo.org/), with the BTRFS file system in encrypted partition. For the task I will use the [Arch Linux](https://archlinux.org/download/) installation media, which contains interesting features that will help a lot in the process. Now, let's get down to business as we have a lot of packages to build... XD

## Disk preparation

This example will use **GPT** as disk partition schema and **grub** as boot loader.

### Create partitions

#### Partitions schema

```bash
/dev/sdX
|--> GRUB BIOS                       2   MB       no fs       grub loader itself
|--> /boot                 boot      512 Mb       fat32       grub and kernel
|--> LUKS encrypted                  100%         encrypted   encrypted binary block 
     |-->  LVM             lvm       100%                  
           |--> /          @root     40 Gb        btrfs       rootfs
           |--> /var       @var      40 Gb        btrfs       var files
           |--> /home      @home     40 GB        btrfs       user files
```

To create **GRUB BIOS**, issue the following command (changing X with your needs):

```bash
parted -a optimal /dev/sdX
```

Set the default units to megabytes:

```bash
unit mib
```

Create a **GPT** partition table:

```bash
mklabel gpt
```

Create the **BIOS** partition:

```bash
mkpart primary 1 3
name 1 grub
set 1 bios_grub on
```

Create **boot** partition. This partition will contain grub files, plain (unencrypted) kernel and kernel initrd:

```bash
mkpart primary fat32 3 515
name 2 boot
set 2 BOOT on
```

Create **logical volume (LVM)** partition:

```bash
mkpart primary 515 -1
name 3 lvm
set 3 lvm on
```

Everything is done, exit from parted:

```bash
quit
```

##### Create boot filesystem

Create filesystem for /dev/sdX2, that will contain grub and kernel files. This partition is read by UEFI bios. Most of motherboards can ready only FAT32 filesystems:

```bash
mkfs.vfat -F32 /dev/sdX2
```

##### Prepare encrypted partition

In the next step, configure DM-CRYPT for /dev/sdX3:
For **Ubuntu live cd**, execute this **command**:

```bash
modprobe dm-crypt
```

Crypt LVM partition /dev/sdX3 with LUKS:

```bash
cryptsetup luksFormat /dev/sdX3
```

Continue by typing in uppercase *YES*.

The following message can be ignored:

```bash
device-mapper: remove ioctl on temporary-cryptsetup-nnnnnn failed: Device or resource busy
```

#### Create LVM inside encrypted block

Open encrypted device (you can change the name '*lvm*'  to one of your choice):

```bash
cryptsetup luksOpen /dev/sdX3 lvm
```

#### Create lvm structure for partition mapping (/root, /var, /home)

Crypt physical volume group:

```bash
lvm pvcreate /dev/mapper/lvm
```

Create volume group vg0  (you can change the name '*vg0*'  to one of your choice):

```bash
vgcreate vg0 /dev/mapper/lvm
```

Create logical volume for /root fs:

```bash
lvcreate -l 100%FREE -n root vg0
```

#### Format partition with BTRFS type and create gentoo dir

```bash
mkfs.btrfs /dev/mapper/vg0-root
mkdir --parents /mnt/gentoo
```

#### Mount partition and create subvolumes

```bash
mount /dev/mapper/vg0-root /mnt
cd /mnt
btrfs subvolume create @
btrfs subvolume create @home
btrfs subvolume create @var
cd && umount /mnt
```

#### Create the directories to mount the partitions

```bash
mkdir -p /mnt/gentoo
mount -o noatime,compress=zstd,space_cache=v2,discard=async,subvol=@ /dev/mapper/vg0-root /mnt/gentoo
mkdir -p /mnt/gentoo/{boot,home,var}
mount -o noatime,compress=zstd,space_cache=v2,discard=async,subvol=@home /dev/mapper/vg0-root /mnt/gentoo/home
mount -o noatime,compress=zstd,space_cache=v2,discard=async,subvol=@var /dev/mapper/vg0-root /mnt/gentoo/var
```

#### Mounting the boot partition

```bash
mount /dev/sdX2 /mnt/gentoo/boot
```

## Getting the files

### Change the current working directory to download the tarball

 ```bash
 cd /mnt/gentoo
 ```

#### Get gentoo stage3

```bash
curl -v https://mirror.bytemark.co.uk/gentoo//releases/amd64/autobuilds/current-stage3-amd64-openrc/stage3-amd64-openrc-20211101T001702Z.tar.xz --output /mnt/gentoo/stage3.tar.xz
```

#### Unzip the downloaded archive

```bash
tar xpvJf stage3.tar.xz --xattrs-include='*.*' --numeric-owner
```

#### Delete unnecessary files

```bash
rm stage3.tar.xz
```

## Adjusting initial settings

### Configure compile options

Open **/mnt/gentoo/etc/portage/make.conf** with nano and setup required flags.

```bash
nano -w /mnt/gentoo/etc/portage/make.conf 
```

#### You can use this file as an example

```bash
# my make.conf
######## Var definitions #######################################################################################
USE_FLAGS="python icu mmx sse sse2 pulseaudio alsa -bindist vaapi vulkan xvmc dbus"
CPU_FLAGS="aes avx avx2 f16c fma3 mmx mmxext pclmul popcnt rdrand sse sse2 sse3 sse4_1 sse4_2 ssse3"
IN_DEVICES="libinput keyboard mouse synaptics evdev"
VIDEO_CARDS="nvidia intel vesa" <--- Choose your video card
EMERGE_OPTS="--ask --autounmask-write=y --with-bdeps=y --quiet-build=y --keep-going=y --jobs 4 --load-average 10.8"
LOCAL="pt-BR en-US"
LICENSES="* @FREE"
BR_MIRRORS="https://gentoo.c3sl.ufpr.br/ http://gentoo.c3sl.ufpr.br/ rsync://gentoo.c3sl.ufpr.br/gentoo/"
################################################################################################################

################################################################################################################
COMMON_FLAGS="-march=skylake -O2 -pipe" <---YOU NEED CHANGE THIS TO YOUR CPU
CFLAGS="${COMMON_FLAGS}"
CXXFLAGS="${COMMON_FLAGS}"
FCFLAGS="${COMMON_FLAGS}"
FFLAGS="${COMMON_FLAGS}"
CPU_FLAGS_X86="$(CPU_FLAGS)"
################################################################################################################

################################################################################################################
MAKEOPTS="-j13 -l12"
EMERGE_DEFAULT_OPTS="${EMERGE_OPTS}"
PORTAGE_NICENESS=19
GRUB_PLATFORMS="efi-64"
VIDEO_CARDS="${VIDEO_CARDS}"
INPUT_DEVICES="${IN_DEVICES}"
AUTOCLEAN="yes"
ACCEPT_LICENSE="${LICENSES}"
ACCEPT_KEYWORDS="~amd64"
L10N="${LOCAL}"
LINGUAS="${LOCAL}"
USE="${USE_FLAGS}"
################################################################################################################

################################################################################################################
# NOTE: This stage was built with the bindist Use flag enabled
PORTDIR="/var/db/repos/gentoo"
DISTDIR="/var/cache/distfiles"
PKGDIR="/var/cache/binpkgs"
################################################################################################################

################################################################################################################
# This sets the language of build output to English.
# Please keep this setting intact when reporting bugs.
LC_MESSAGES=C
################################################################################################################

################################################################################################################
GENTOO_MIRRORS="${BR_MIRRORS}"
################################################################################################################
```

## Preparing to chroot

### Configuring the repository

This can be done in a few simple steps. First, if it does not exist, create the **repos.conf** directory:

```bash
mkdir --parents /mnt/gentoo/etc/portage/repos.conf
```

Next, copy the Gentoo repository configuration file provided by Portage to the (newly created) **repos.conf** directory:

```bash
cp /mnt/gentoo/usr/share/portage/config/repos.conf /mnt/gentoo/etc/portage/repos.conf/gentoo.conf
```

## DNS

Copy DNS info:

```bash
cp --dereference /etc/resolv.conf /mnt/gentoo/etc/
```

## Mounting system files

Mounting the necessary filesystems:

```bash
mount --types proc /proc /mnt/gentoo/proc
mount --rbind /sys /mnt/gentoo/sys
mount --make-rslave /mnt/gentoo/sys
mount --rbind /dev /mnt/gentoo/dev
mount --make-rslave /mnt/gentoo/dev
mount --bind /run /mnt/gentoo/run
mount --make-slave /mnt/gentoo/run
```

### Warning


When using **non-Gentoo installation media**, this might not be sufficient. Some distributions make **/dev/shm** a symbolic link to **/run/shm/** which, after the **chroot**, becomes invalid. Making **/dev/shm/ a proper tmpfs** mount up front can fix this:

```bash
test -L /dev/shm && rm /dev/shm && mkdir /dev/shm
mount -t tmpfs -o nosuid,nodev,noexec shm /dev/shm
```

Also ensure that **mode 1777** is set:

```bash
chmod 1777 /dev/shm
```

### Gensftab script

If you are using the arch-iso to installation, you can use the **genfstab** script to generate the fstab file automatically.

```bash
genfstab -U /mnt/gentoo >> /mnt/gentoo/etc/fstab
```

## Entering the new environment

### Chroot

```bash
chroot /mnt/gentoo /bin/bash
source /etc/profile
export PS1="(chroot) $PS1"
```

### Portage sync

#### Install files

```bash
emerge-webrsync && emerge --sync
```

#### Profile selection

Choose and install correct profile (8 is a desktop with plasma):

```bash
eselect profile list
eselect profile set 8
```

Setup correct timezone:

```bash
echo America/Sao_Paulo > /etc/timezone
emerge --config sys-libs/timezone-data
```

Configure locales

```bash
nano -w /etc/locale.gen
locale-gen
```

Set default locale

```bash
eselect locale list
eselect locale set X
```

Update env:

```bash
env-update && source /etc/profile && export PS1="(chroot) ${PS1}"
```

## Configure FSTAB

 For consistent setup of the required partition, use the UUID identifyer.

Run ```blkid``` and see partition IDs:

```bash
blkid
/dev/sdb1: UUID="4F20-B9DB" TYPE="vfat" PARTLABEL="grub" PARTUUID="70b1627b-57e7-4559-877a-355184f0ab9d"
/dev/sdb2: UUID="DB1D-89C5" TYPE="vfat" PARTLABEL="boot" PARTUUID="b2a61809-4c19-4685-8875-e7fdf645eec5"
/dev/sdb3: UUID="6a7a642a-3262-4f87-9540-bcd53969343b" TYPE="crypto_LUKS" PARTLABEL="lvm" PARTUUID="be8e6694-b39c-4d2f-9f42-7ca455fdd64f"
/dev/mapper/lvm: UUID="HL32bg-ZjrZ-RBo9-PcFM-DmaQ-QbrC-9HkNMk" TYPE="LVM2_member"
/dev/mapper/vg0-root: UUID="6bedbbd8-cea9-4734-9c49-8e985c61c120" TYPE="ext4"
/dev/mapper/vg0-var: UUID="61e4cc83-a1ee-4190-914b-4b62b49ac77f" TYPE="ext4"
/dev/mapper/vg0-home: UUID="5d6ff087-50ce-400f-91c4-e3378be23c00" TYPE="ext4"
```

### Edit /etc/fstab

Example of fstab

```bash
# NOTE: If your BOOT partition is ReiserFS, add the notail option to opts.
#
# NOTE: Even though we list ext4 as the type here, it will work with ext2/ext3
#       filesystems.  This just tells the kernel to use the ext4 driver.
#
# NOTE: You can use full paths to devices like /dev/sda3, but it is often
#       more reliable to use filesystem labels or UUIDs. See your filesystem
#       documentation for details on setting a label. To obtain the UUID, use
#       the blkid(8) command.

# device/UUID                             mounting point     fs_type           options                                                                          0 0

# /dev/sdX2
UUID=D85D-FE1F                                  /boot           vfat            defaults                                                                        0 2
# /dev/mapper/vg0-root
UUID=1ea2dfd2-a6d7-43fe-b8af-a5364bf0b486       /               btrfs           rw,noatime,compress=zstd:3,discard=async,space_cache,subvolid=256,subvol=/@     0 0

# /dev/mapper/vg0-root
UUID=1ea2dfd2-a6d7-43fe-b8af-a5364bf0b486       /home           btrfs           rw,noatime,compress=zstd:3,discard=async,space_cache,subvolid=258,subvol=/@home 0 0

# /dev/mapper/vg0-root
UUID=1ea2dfd2-a6d7-43fe-b8af-a5364bf0b486       /var            btrfs           rw,noatime,compress=zstd:3,discard=async,space_cache,subvolid=259,subvol=/@var  0 0

# tracefs
tracefs                                 /sys/kernel/tracing     tracefs         rw,nosuid,nodev,noexec                                                                          0 0
```

## Kernel

### Configuration and compilation

Install kernel, genkernel and cryptsetup packages:

```bash
emerge --ask sys-kernel/linux-firmware sys-firmware/intel-microcode
emerge sys-kernel/gentoo-sources
```

Use **eselect** to display available kernel list:

```bash
eselect kernel list
Available kernel symlink targets:
  [1]   linux-5.15.0-gentoo
```

In order to create a symbolic link called linux, use:

```bash
eselect kernel set 1 && eselect kernel list
Available kernel symlink targets:
  [1]   linux-5.15.0-gentoo *

ls -l /usr/src/linux
lrwxrwxrwx 1 root root 19 Nov  1 16:58 /usr/src/linux -> linux-5.15.0-gentoo
```

### Using genkernel

Install genkernel:

```bash
emerge --ask sys-apps/pciutils
emerge --ask sys-kernel/genkernel
emerge sys-fs/cryptsetup
```

#### Warning: before compile the kernel , check your fstab configuration

Build genkernel

```bash
genkernel --btrfs --luks --lvm --no-zfs --microcode all
```

## Network config

### Networking information

#### Host and domain information:

```bash
nano -w /etc/conf.d/hostname
# Set the hostname variable to the selected host name
hostname="gentoo"
```

```bash
nano -w /etc/conf.d/net
# Set the dns_domain_lo variable to the selected domain name
config_enp2s0="dhcp"
dns_domain_lo="homenetwork"
```

### Automatically start networking at boot

```bash
emerge --ask --noreplace net-misc/netifrc
cd /etc/init.d
ln -s net.lo net.enp2s0
rc-update add net.enp2s0 default
```

#### Configure hosts

```bash
nano -w /etc/hosts
# This defines the current system and must be set
127.0.0.1     gentoo.homenetwork gentoo localhost
```

####Get PMCIA works

```bash
emerge --ask sys-apps/pcmciautils
```

## System information

### Change root password

Set the root password

```bash
passwd
```

The root Linux account is an all-powerful account, so pick a strong password. Later an additional regular user account will be created for daily operations.

### Init and boot configuration

Gentoo (at least when using OpenRC) uses **/etc/rc.conf** to configure the services, startup, and shutdown of a system. Open up /etc/rc.conf and enjoy all the comments in the file. Review the settings and change where needed.

```bash
root #nano -w /etc/rc.conf
```

### Keyboard layout

Next, open **/etc/conf.d/keymaps** to handle keyboard configuration. Edit it to configure and select the right keyboard.

```bash
nano -w /etc/conf.d/keymaps
```

Take special care with the keymap variable. If the wrong keymap is selected, then weird results will come up when typing on the keyboard.

## System logger

### Using sysklogd and cronie

```bash
emerge --ask app-admin/sysklogd
emerge --ask sys-process/cronie
```

#### Adding to default runlevel

```bash
rc-update add sysklogd default
rc-update add cronie default
```

#### Optional: File indexing

```bash
emerge --ask sys-apps/mlocate
```

#### Optional: Remote access

```bash
rc-update add sshd default
```

## Filesystem tools

Add the ones you want to your system or install them all if you want

```bash
emerge --ask sys-fs/e2fsprogs
emerge --ask sys-fs/xfsprogs
emerge --ask sys-fs/reiserfsprogs
emerge --ask sys-fs/jfsutils
emerge --ask sys-fs/dosfstools
emerge --ask sys-fs/btrfs-progs
emerge --ask sys-fs/zfs
```

## Networking tools

### Install ethernet tools

```bash
emerge --ask net-misc/dhcpcd
```

### Install wireless networking tools

```bash
emerge --ask net-wireless/wpa_supplicant
```

If you get an error of circular dependencies when trying to install wpa_supplicant, try this command below:

```bash
USE="-X -cairo -glib -graphite -harfbuzz -icu -introspection -png -truetype" emerge media-libs/freetype media-libs/harfbuzz
```

## GRUB2

### Warning
Don't forget to change "REPLACE ME WITH sdX3 UUID"  to the actual value.

```bash
nano /etc/default/grub
GRUB_CMDLINE_LINUX="dolvm crypt_root=UUID=REPLACE ME WITH sdX3 UUID"
```

```bash
echo "sys-boot/grub:2 device-mapper" >> /etc/portage/package.use/sys-boot
emerge --ask --verbose sys-boot/grub:2
```

### Mount boot partition

```bash
mount /boot
```

A note for UEFI users: running the above command will output the enabled GRUB_PLATFORMS values before emerging. When using UEFI capable systems, users will need to ensure GRUB_PLATFORMS="efi-64" is enabled (as it is the case by default). If that is not the case for the setup, GRUB_PLATFORMS="efi-64" will need to be added to the /etc/portage/make.conf file before emerging GRUB2 so that the package will be built with EFI functionality:

### Install GRUB2

```bash
grub-install --target=x86_64-efi --efi-directory=/boot
```

### Start LVM services

```bash
rc-update add lvm default
```

### Configure GRUB2

```bash
grub-mkconfig -o /boot/grub/grub.cfg
Generating grub.cfg ...
Found linux image: /boot/vmlinuz-5.15.0-gentoo
Found initrd image: /boot/initramfs-genkernel-amd64-5.15.0-gentoo
done
```

#### Rebooting the system

Exit the chrooted environment and unmount all mounted partitions. Then type in that one magical command that initiates the final, true test: reboot.

```bash
exit
cd
umount -l /mnt/gentoo/dev{/shm,/pts,}
umount -R /mnt/gentoo
reboot
```
