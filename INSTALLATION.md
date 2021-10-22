# Installation

pacstrap -i /mnt base base-devel linux-firmware linux-hardened linux man-db man-pages intel-ucode

arch-chroot /mnt

pacman -S linux-headers linux-hardened-headers

pacman -S lvm2 texinfo nano git curl wget

## Networking 

pacman -S openssh networkmanager wpa_supplicant wireless_tools dialog openvpn networkmanager-openvpn networkmanager-vpnc network-manager-applet dhclient libsecret nm-connection-editor modemmanager usb_modeswitch rp-ppoe networkmanager-openconnect networkmanager-pptp networkmanager-l2tp iptables

systemctl enable NetworkManager

## LVM Support

nano /etc/mkinitcpio.conf 

    HOOKS....block encrypt lvm2 filesystem

mkinitcpio -p linux

mkinitcpio -p linux-hardened

## Timezone 

timedatectl set-timezone America/Denver

hwclock -systohc

systemctl enable systemd-timesyncd

## generate locales

nano /etc/locale.gen

    en_us.UTF-8 UTF-8
    en_us ISO-8859-1

locale-gen

## Hostname

hostnamectl set-hostname $(HOSTNAME) <!--- hostname-->

nano /etc/hosts

    127.0.0.1 localhost
    ::1 localhost
    127.0.1.1 $(HOSTNAME).localdomain $(HOSTNAME)

## Setup Root and User

passwd
>enter root password

useradd -m -g users -G wheel $(USER) <!--- Setup User -->

passwd $(USER)
>enter user password

EDITOR=nano visudo <!--- Add user to sudo -->
    %wheel ALL=(ALL) ALL

## Yaourt 
cd /opt

sudo git clone https://aur.archlinux.org/yay.git

sudo chown -R $(USER):wheel ./yay
cd yay

makepkg -si

cd

# encrypted booting from UEFI directly

pacman -S efitools efivar sbsigntools python python-pip python2 python2-pip efibootmgr openssl

blkid /dev/sda2 > /boot/boot.txt

nano /boot/boot.txt

    cryptdevice=UUID=$(UUID):$(DRIVE):allow-discards root=/dev/mapper/$(DRIVENAME)-root rw quiet lsm=lockdown,yama,apparmor,bpf i915.fastboot=1

cd /boot
    objcopy \
        --add-section .osrel=/etc/os-release --change-section-vma .osrel=0x20000 \ --add-section .cmdline="boot.txt" --change-section-vma .cmdline=0x30000 \ -add-section .linux="vmlinuz-linux-hardened" --change-section-vma .linux=0x40000 \ --add-section .initrd="initramfs-linux-hardened.img" --change-section-vma .initrd=0x3000000 \ /usr/lib/systemd/boot/efi/linuxx64.efi.stub /efi/BOOT/Arch/linux-hardened-signed.efi

### keys
yay -S sbupdate-git sbkeys
<!--- Make Sure in /root --->

mkdir /etc/efi-keys

cd !$

wget https://www.rodsbooks.com/efi-bootloaders/mkkeys.sh

chmod +x mkkeys.sh

./mkkeys.sh 

<enter a the key db name >

### modify sbupdate

nano /etc/sbupdate.conf

    ESP_DIR="/efi"

    OUT_DIR="BOOT/Arch"

    CONFIG

    CMDLINE_DEFAULT="cryptdevice=UUID=$(UUID):$(DRIVENAME):allow-discards root=/dev/mapper/$(DRIVENAME)-root rw quiet"


#### manually sign kernel

    sbsign --key /etc/efi-keys/DB.key --cert /etc/efi-keys/DB.crt --output /efi/BOOT/Arch/linux-hardened-signed.efi /efi/BOOT/Arch/linux-hardened-signed.efi

## linux bootloader

efibootmgr -c -d /dev/sda -p 1 --label "BunnyArch" -l "BOOT\Arch\linux-hardened-signed.efi" --verbose


#### check if valid boot
efibootmgr 

verify BunnyArch is first, if not reorder by:

efibootmgr -o 0002,2001,2020 with the corresponding boot numbers of your list.

https://github.com/theo546/my-arch-setup#readme

https://pawitp.medium.com/full-disk-encryption-on-arch-linux-backed-by-tpm-2-0-c0892cab9704

https://gist.github.com/huntrar/e42aee630bee3295b2c671d098c81268

https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system#Encrypted_boot_partition_(GRUB)

https://bentley.link/secureboot/

https://webhamster.ru/mytetrashare/index/mtb375/1606750057ix5fawcxkn

https://github.com/andreyv/sbupdate

https://bentley.link/secureboot/

https://wiki.archlinux.org/title/Systemd-boot#Configuration

https://wiki.archlinux.org/title/Sysctl#TCP.2FIP_stack_hardening

https://wiki.archlinux.org/index.php/security#Restricting_root

https://wiki.archlinux.org/index.php/Category:Firewalls
