# Arch install btrfs with encrypted disk
iwctl station wlan0 connect _ 

exit 
 
systemctl start sshd.service 
 
passwd 
 
ip addr show 

#### Now, on the second PC, call into the first PC via SSH:
> ssh root@'IP-OF-THE-FIRST-PC'

timedatectl set-ntp true

cryptsetup luksFormat --align-payload=8192 -c aes-xts-plain64 -h sha512 -s 512 --use-random /dev/nvme0n1p2 

cryptsetup open /dev/nvme0n1p2 nene 

mkfs.vfat -F32 -n GIT /dev/nvme0n1p1 

mkfs.btrfs -L .trash /dev/mapper/nene 

mount /dev/mapper/nene /mnt 


btrfs subvolume create /mnt/@ && btrfs subvolume create /mnt/@home && btrfs subvolume create /mnt/@swap && btrfs subvolume create /mnt/@pkg && btrfs subvolume create /mnt/@snapshots 

umount /mnt 

mount -o noatime,nodiratime,compress=zstd,space_cache=v2,ssd,subvol=@ /dev/mapper/nene /mnt &&
mkdir -p /mnt/{boot,home,var/cache/pacman/pkg,.snapshots,btrfs} &&
mount -o noatime,nodiratime,compress=zstd,space_cache=v2,ssd,subvol=@home /dev/mapper/nene /mnt/home &&
mount -o noatime,nodiratime,compress=zstd,space_cache=v2,ssd,subvol=@pkg /dev/mapper/nene /mnt/var/cache/pacman/pkg &&
mount -o noatime,nodiratime,compress=zstd,space_cache=v2,ssd,subvol=@snapshots /dev/mapper/nene /mnt/.snapshots &&
mount -o noatime,nodiratime,compress=zstd,space_cache=v2,ssd,subvolid=5 /dev/mapper/nene /mnt/btrfs 

mount /dev/nvme0n1p1 /mnt/boot

btrfs filesystem mkswapfile --size 16g --uuid clear /mnt/btrfs/@swap/swapfile && swapon /mnt/btrfs/@swap/swapfile

reflector --latest 5 --sort rate --save /etc/pacman.d/mirrorlist 

pacman-key --init && pacman -Sy archlinux-keyring

pacstrap -K /mnt base base-devel btrfs-progs intel-ucode mkinitcpio-nfs-utils xf86-video-intel xorg-server xf86-input-libinput linux-firmware sof-firmware linux-firmware-qlogic fwupd wireless-regdb networkmanager terminus-font man-db man-pages texinfo git vim plymouth snapper pam-u2f macchanger openssh plasma libreoffice git docker python

genfstab -U /mnt >> /mnt/etc/fstab

arch-chroot /mnt

# User creation
```
useradd -s /bin/bash -m -G wheel -G docker me 

passwd me 

visudo #uncomment %wheel ALL=(ALL) ALL 

su -l me 


git clone https://aur.archlinux.org/ast-firmware.git && cd ast-firmware && makepkg -Ccris && git clone https://aur.archlinux.org/aic94xx-firmware.git && cd aic94xx-firmware && makepkg -Ccris && git clone https://aur.archlinux.org/wd719x-firmware.git && cd wd719x-firmware && makepkg -Ccris && git clone https://aur.archlinux.org/upd72020x-fw.git && cd upd72020x-fw && makepkg -Ccris 

cd ../../../..

rm -rf ast-firmware 

exit 
```

# Install liquorix kernel

> curl 'https://liquorix.net/install-liquorix.sh' | bash 

Or install the official linux-zen with pacman -Syu linux-zen, if liquorix is out.

# Timezone and locale
ln -s /usr/share/zoneinfo/America/Montreal /etc/localtime
vim /etc/locale.gen (uncomment en_US.UTF-8 UTF-8)
echo LANG=en_US.UTF-8 >> /etc/locale.conf

locale-gen

# Set keyboard configs
echo KEYMAP=us >> /etc/vconsole.conf
echo FONT=default8x16 >> /etc/vconsole.conf

hwclock --systohc

# Hostname config
echo host >> /etc/hostname
/etc/hosts
```
127.0.0.1	localhost
::1		localhost
127.0.1.1	host.localdomain	host
```

vim /etc/mkinitcpio.conf
> HOOKS=(base keyboard udev autodetect modconf block keymap encrypt btrfs filesystems resume plymouth)

mkinitcpio -p linux-lqx (Or mkinitcpio -p linux-zen)

bootctl --path=/boot install

#### For liquorix /boot/loader/entries/dawrlog.conf
```
title Dawrlog
linux /vmlinuz-linux-lqx
initrd /intel-ucode.img
initrd /initramfs-linux-lqx.img
``` 

#### For linux zen /boot/loader/entries/dawrlog.conf
```
title Dawrlog
linux /vmlinuz-linux-zen
initrd /intel-ucode.img
initrd /initramfs-linux-zen.img
``` 

Then update the btrfs from your configured device 

```
echo "options cryptdevice=UUID=$(blkid -s UUID -o value /dev/nvme0n1p2):nene:allow-discards root=/dev/mapper/nene rootflags=subvol=@ rd.luks.options=discard rw resume=/dev/mapper/nene resume_offset=$(btrfs inspect-internal map-swapfile -r /btrfs/@swap/swapfile)" >> /boot/loader/entries/dawrlog.conf
```

/boot/loader/loader.conf
```
default  dawrlog.conf
timeout  4
console-mode max
editor   no
```

/usr/share/plymouth/plymouthd.defaults
```
[Daemon]
Theme=dawrlog
ShowDelay=5
DeviceTimeout=8
```

```
cd ~
git clone https://github.com/dawrlog/goodies.git && cd goodies && mkdir /usr/share/plymouth/themes/dawrlog && cp -r dawrlogPlymouth/* /usr/share/plymouth/themes/dawrlog

exit
swapoff /mnt/btrfs/@swap/swapfile
umount -R /mnt
reboot

nmcli device wifi connect <AP name> password <AP pwd>

ip addr show
```

#### ssh root@<IP-OF-THE-FIRST-PC>

```
fwupdmgr refresh
fwupdmgr get-updates
fwupdmgr update

echo "[multilib]" >> /etc/pacman.conf
echo "Include = /etc/pacman.d/mirrorlist" >> /etc/pacman.conf

curl 'https://blackarch.org/strap.sh' | bash
curl 'https://raw.githubusercontent.com/dawrlog/goodies/main/wkfilters.sh' | bash

pacman -Syu tlp touchegg fprintd sddm wireshark-qt wireshark-cli nmap kde-applications #8 18 19 23 38 53 58 59 123 144 171 176

curl https://raw.githubusercontent.com/dawrlog/goodies/main/macchanger.conf >> /etc/systemd/system/macspoof@wlp0s20f3.service && systemctl enable macspoof@wlp0s20f3.service

git clone https://aur.archlinux.org/brave-bin.git && cd brave-bin && makepkg -Ccris

```

sudo systemctl enable fprintd
