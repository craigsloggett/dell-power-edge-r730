# Dell PowerEdge R730
A single place to put everything related to my server.

---

## Preparing the Primary Disk

I typically boot into an Arch Linux installation image since it has all of the commands available
needed to prepare the disk.

### Removing State

Before setting up a new Operating System on my laptop, I like to ensure that the stateful components
like the disk and EFI boot entries are wiped clean.

The tools used to accomplish this are `blkdiscard(8)` and `efibootmgr(8)`.

Unfortunately, the SSD currently installed in the server does not support `BLKSECDISCARD` commands
to securely wipe the device. To zero out and wipe the SSD device,

```shell
dd if=/dev/zero of=/dev/sdX bs=1024k count=128; sync
blkdiscard /dev/sdX
```

To remove all EFI boot entries,

```shell
efibootmgr -b XXXX -B
```

### Prepare the Disk (Unencrypted)

Using `cfdisk /dev/sdX`, I have partitioned my disk as follows:

| Partition     | Size   | Type             | Format |
|---------------|--------|------------------|--------|
| /dev/sdX1     | 512M   | EFI System       | FAT32  |
| /dev/sdX2     | 465.3G | Linux Filesystem | BTRFS  |

Format the new partitions:

```shell
mkfs.fat -F32 /dev/sdX1
mkfs.btrfs    /dev/sdX2
```

Getting ready for a new OS installation:

```shell
mount /dev/sdX2 /mnt
mkdir -p /mnt/boot
mount /dev/sdX1 /mnt/boot
```

Create a 16G swapfile in the `/var` directory:

```shell
mkdir -p /mnt/var
dd if=/dev/zero of=/mnt/var/swapfile bs=4k count=4M
chmod 600 /mnt/var/swapfile
mkswap -U clear /mnt/var/swapfile
```

## Install Arch Linux

### Sort Mirror List

```shell
url="https://www.archlinux.org/mirrorlist/?country=CA&protocol=https&ip_version=4&use_mirror_status=on"
tmpfile=$(mktemp --suffix=-mirrorlist)
curl -L "${url}" -o "${tmpfile}"
cat "${tmpfile}" >> /etc/pacman.d/mirrorlist
vim /etc/pacman.d/mirrorlist  # Uncomment and put the closest mirrors at the top of the list.
```

### Install Base Packages

```shell
pacstrap /mnt base linux linux-firmware
```

### Mount Storage

```shell
mkdir -p /mnt/srv/storage
mount /dev/sdX1 /mnt/srv/storage
```

### Configure fstab

```shell
genfstab -U /mnt >> /mnt/etc/fstab
```

Add the swapfile line to `/mnt/etc/fstab` (not added by `genfstab`),

```shell
# Swapfile
/var/swapfile   none    swap    defaults    0 0
```

### Configure the Timezone

```shell
arch-chroot /mnt
ln -sf /usr/share/zoneinfo/Canada/Eastern /etc/localtime
hwclock --systohc
exit
```

### Enable the Time Sync Service

```shell
arch-chroot /mnt
systemctl enable systemd-timesyncd.service
exit
echo '[Time]' > /mnt/etc/systemd/timesyncd.conf
echo 'NTP=0.ca.pool.ntp.org 1.ca.pool.ntp.org 2.ca.pool.ntp.org 3.ca.pool.ntp.org'                 >> /mnt/etc/systemd/timesyncd.conf
echo 'FallbackNTP=0.arch.pool.ntp.org 1.arch.pool.ntp.org 2.arch.pool.ntp.org 3.arch.pool.ntp.org' >> /mnt/etc/systemd/timesyncd.conf
```

### Configure the System Locale

```shell
vim /mnt/etc/locale.gen  # Uncomment en_CA.UTF-8
arch-chroot /mnt
locale-gen
exit
echo 'LANG=en_CA.UTF-8' > /mnt/etc/locale.conf
```

### Configure the Hostname

```shell
echo 'saturn' > /mnt/etc/hostname
```

### Configure the Network

```shell
arch-chroot /mnt
systemctl enable systemd-networkd.service
systemctl enable systemd-resolved.service
exit
echo "[Match]"    > /mnt/etc/systemd/network/10-wired.network
echo "Name=eno1" >> /mnt/etc/systemd/network/10-wired.network
echo ""          >> /mnt/etc/systemd/network/10-wired.network
echo "[Network]" >> /mnt/etc/systemd/network/10-wired.network
echo "DHCP=yes"  >> /mnt/etc/systemd/network/10-wired.network
rm /etc/resolv.conf
ln -rsf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
```

### Configure a Regular User

```shell
arch-chroot /mnt
useradd -m -G wheel -s /bin/bash <user>
passwd <user>  # Set the password for the regular user
exit
```

### Configure the EFI Bootloader

```shell
arch-chroot /mnt
bootctl --path=/boot install
exit

echo 'timeout 4' >> /mnt/boot/loader/loader.conf
echo 'editor  0' >> /mnt/boot/loader/loader.conf

echo 'title     Arch Linux'            > /mnt/boot/loader/entries/archlinux.conf
echo 'linux     /vmlinuz-linux'       >> /mnt/boot/loader/entries/archlinux.conf

# Setup Intel microcode updates.
echo 'initrd    /intel-ucode.img'     >> /mnt/boot/loader/entries/archlinux.conf

# Linux initramfs must go after any microcode updates.
echo 'initrd    /initramfs-linux.img' >> /mnt/boot/loader/entries/archlinux.conf
echo 'options   root=/dev/sda2 rw'    >> /mnt/boot/loader/entries/archlinux.conf

# Update the bootloader
arch-chroot /mnt
bootctl update;
exit
```

### Install Intel Microcode

```shell
pacstrap /mnt intel-ucode
```

### Set the root User Password

```shell
arch-chroot /mnt
passwd  # Set the password for the root user
```

### Reboot

```shell
reboot
```

## Post Installation

### Enable NTP

```shell
timedatectl set-ntp true
```

### Install vi

```shell
pacman -Syu vi
```

### Install SSH

```shell
pacman -Syu openssh
```

#### Configure SSH

`/etc/ssh/sshd_config`

```
AddressFamily inet
PermitRootLogin no
PasswordAuthentication no
```

```shell
systemctl enable --now sshd.service
```

### Install Git

```shell
pacman -Syu git
```

### Install the Man Pages

```shell
pacman -Syu man-db man-pages
```

## NFS

### Prepare the Share

Bind mount the storage directory to a directory to be used as the NFS share,

```shell
mkdir -p /srv/nfs/media
mount --bind /srv/storage/media /srv/nfs/media
```

Add the bind mount to `/etc/fstab` to ensure it persists across reboots,

**/etc/fstab**
```
/srv/storage/media  /srv/nfs/media  none  bind  0 0
```

### Install the NFS Package

```shell
pacman -Syu nfs-utils
```

### Configure NFS Exports

**/etc/exports**
```
/srv/nfs        192.168.1.0/24(ro,sync,no_subtree_check,crossmnt,fsid=0)
/srv/nfs/media  192.168.1.0/24(ro,sync,no_subtree_check)
```

```shell
systemctl enable --now nfs-server.service
```

## Containers

### Install the Arch Install Scripts

```shell
pacman -Syu arch-install-scripts
```

### Bootstrap the Filesystem

```shell
mkdir -p ~/my-container
pacstrap -K -c ~/my-container/ base
```

### Import the Machine

```shell
machinectl import-fs ~/my-container/ my-container
```

### Start the Machine

```shell
machinectl start my-container
```

### Launch a Shell in the Machine

```shell
machinectl shell my-container
```

### Configure a Regular User

```shell
useradd -m -G wheel -s /bin/bash <user>
passwd <user>
```

### Set the root Password

```shell
passwd
```

### Configure Networking

```shell
systemctl enable --now systemd-networkd.service
systemctl enable --now systemd-resolved.service
exit
```

### Bind Mount Local Storage

**/etc/systemd/nspawn/my-container.nspawn**
```shell
[Files]
Bind=/srv/storage/media:/srv/storage/media:idmap
```

### Stop the Machine

```shell
machinectl stop my-container
```

### Enable the Machine to Start at Boot

```shell
systemctl enable machines.target
machinectl enable my-container
```

### Start and Login to the Machine

```shell
machinectl start my-container
machinectl login my-container
```
