# arch-build
## Overview
This walkthrough is for setting up a standard EFI boot machine. It does not include any raid or disk encryption. Setup includes btrfs subvolumes with snapshot backups and a swapfile instead of the usual swap partition (they are virtualy the same but a swapfile is much easier to modify incase you wish to change something). Setup only includes the base system without a desktop enviornment.
## setup FS
First find your connected storage devices with `lsblk -df`, take a photo (or screenshot, if your using a VM) for future refrence.
Then find the one you want to partition with `udevadm info /dev/<device>` this tells you the devices model info.

```
cfdisk /dev/<drive>
```
> Create a partition of 1G, this will be your boot partition. Format it with the next command.
```  
mkfs.fat -F32 /dev/<drive>
```
> This formats the partition to fat32. This is not the only filesystem type you can boot from but is the easiest to work with and is the most well supported.
```
mkfs.btrfs -L <lable> -f -M --csum sha256 -O quota /dev/sdX
```
> 
```
mount /dev/<drive> /mnt
```
```
btrfs subvolume create /mnt/@
```
> This is Your file system root indicated by the "`/`"
```
  btrfs subvolume create /mnt/@home
```
> Your `/home` partition. This is where all users home folders are stored with the exeption of "root" user (the system admin) of which can access all data on the machine despite who owns it. "root" owns **EVERYTHING**!
```
    btrfs subvolume create /mnt/@log
```
```
    btrfs subvolume create /mnt/@pkg
```
```
    btrfs subvolume create /mnt/@swap
```
```
btrfs subvolume create /mnt/@snapshots
```
### Mount subvols

> First unmount /mnt

    umount /mnt
<br>

> sets up @ as root subvol

    mount -o subvol=@,compress=zstd,noatime /dev/sdXX /mnt
<br>

> Creates the Dirs the subvols will be mounted to

    mkdir -p /mnt/{home,boot,var/log,var/cache/pacman/pkg,.swap,.snapshots}
<br>

    mount -o subvol=@home,compress=zstd,noatime /dev/sdXX /mnt/home
<br>

    mount -o subvol=@log,compress=zstd,noatime /dev/sdXX /mnt/var/log
<br>

    mount -o subvol=@pkg,compress=zstd,noatime /dev/sdXX /mnt/var/cache/pacman/pkg
<br>

    mount -o subvol=@swap,nodatacow,noatime /dev/sdXX /mnt/.swap
<br>

    mount -o subvol=@snapshots,compress=zstd,noatime /dev/sdXX /mnt/.snapshots
<br>

> Mounting EFI Boot Part

    mount /dev/sdXX /mnt/boot
<br>

### Time to Verify!

> lists all the mounted filesys's and subvol. (you could use `lsblk -f` but this has slightly easier to read formating)

    findmnt -R /mnt
<br>

## INSTALL THE SYSTEM!

    pacstrap -K /mnt base btrfs-progs amd-ucode sudo nano linux-zen linux-lts linux-firmare scx-scheds wireless-regdb dracut binutils elfutils networkmanager squashfs-tools systemd-ukify tpm2-tools sbsigntools cryptsetup rng-tools qrencode jq nvme-cli dbus-broker dbus bluez openssh plymouth tuned-ppd wireless_tools systemtap firewalld 
<br>

### Gen Fstab!

    genfstab -U /mnt >> /mnt/etc/fstab
<br>

### CHROOT!

    arch-chroot -S /mnt
<br>

    nano /etc/dracut.conf.d/main.conf
<br>

    hostonly="yes"
    compress="zstd"
    add_dracutmodules+=" tpm2-tss crypt plymouth bluetooth "
<br>

    blkid -s UUID -o value /dev/sdXX   # Your btrfs partition UUID
<br>

    nano /etc/dracut.conf.d/cmdline.conf
<br>

    kernel_cmdline="root=UUID=<YOUR-UUID-HERE> rootfstype=btrfs rootflags=subvol=@ rw quiet splash"
<br>

    nano /etc/dracut.conf.d/i18n.conf
<br>

    i18n_vars="KEYMAP=us"
<br>

     bootctl install
<br>

     nano /boot/loader/loader.conf
<br>
     
    default  linux.conf
    timeout  3
    console-mode auto
    editor   no
<br>

    nano /boot/loader/entries/linux.conf
<br>

    title   Linux
    linux   /vmlinuz-linux-zen
    initrd  /initramfs-linux-zen.img
<br>

    nano /boot/loader/entries/linux-fallback.conf
<br>

    title   Fallback (LTS)
    linux   /vmlinuz-linux-lts
    initrd  /initramfs-linux-lts-fallback.img
<br>

    dracut -f --regenerate-all
<br>

    ln -sf /usr/share/zoneinfo/Region/City /etc/localtime
<br>

    hwclock --systohc
<br>

    nano /etc/locale.gen
<br>

    locale-gen
<br>
    
    echo "LANG=en_US.UTF-8" > /etc/locale.conf
<br>

    echo "yourhostname" > /etc/hostname
<br>

    sudo nano /etc/conf.d/wireless-regdom
<br>

    passwd
<br>

    useradd -m -s /bin/bash -G wheel -c “<full name(optional)>” <username>
<br>

    passwd <username>
<br>

> at the bottom uncomment (remove the leading `#`) the line `%wheel ALL=(ALL:ALL) ALL`

    EDITOR=nano visudo
<br>

    exit
<br>

    umount -R /mnt
<br>

    reboot
    
