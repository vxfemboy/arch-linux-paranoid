# Bunny Arch Linux
### An Arch Linux-based security, development, and pentesting distribution to be used by tech-savvy devs, ethical hackers, and more; who perfer simplicity and automation with the easy things and handling the advanced.

## UFEI Linux Hardened w/ Encryption
Follow Security protocols

https://github.com/trimstray/linux-hardening-checklist

https://github.com/trimstray/the-practical-linux-hardening-guide

https://wiki.archlinux.org/title/Security

https://wiki.archlinux.org/title/Iptables

https://wiki.archlinux.org/title/SELinux
s
https://wiki.archlinux.org/title/SELinux#Enable_SELinux_LSM

https://wiki.archlinux.org/title/Systemd-boot#Automatic_update

https://wiki.archlinux.org/title/Kernel_parameters

# DISK SETUP

fdisk -l

fdisk /dev/sda <!--- or users drive -->

## Hardening Checks

g <!--- new partition table -->

n
⏎ 
⏎ 
+500M <!--- for efi -->
⏎ 
t
⏎ 
1

n
⏎ 
2
⏎ 
+500M

n
⏎ 
3
⏎
⏎ 
t
⏎ 
30 <!--- for LVM -->
⏎ 
w

## 3 partions time for formating
mkfs.fat -F32 /dev/sda1 

mkfs.ext4 /dev/sda2

## time to encrypt
cryptsetup luksFormat /dev/sda3 

YES
> enter encryption passphrase

cryptsetup open --type luks /dev/sda3 lvm 
> enter encryption passphrase

## setup lvm, 'physical' volume, and logical volumes

pvcreate --dataalignment 1m /dev/mapper/lvm

vgcreate $(DRIVENAME) /dev/mapper/lvm <!--- Volume Name -->

lvcreate -L 100GB $(DRIVENAME) -n root <!--- Storage for /root --> 

lvcreate -L 10GB $(DRIVENAME) -n var <!---storage for /var -->

lvcreate -L 5GB $(DRIVENAME) -n log <!---storage for /var/log & /var/log/audit -->

lvcreate -L 5GB $(DRIVENAME) -n tmp <!--- Storage for /tmp & /var/tmp -->

lvcreate -L 20GB $(DRIVENAME) -n usr <!---storage for /usr -->

lvcreate -l 100%FREE $(DRIVENAME) -n home <!--- Storage for /home --> 

## activate volumes
modprobe dm_mod

vgscan

vgchange -ay

## format & mount volumes

### /root
mkfs.ext4 /dev/$(DRIVENAME)/root

mount /dev/$(DRIVENAME)/root /mnt

### /boot

mkdir /mnt/boot

mount /dev/sda2 /mnt/boot

### /var

mkfs.ext4 /dev/$(DRIVENAME)/var

mkdir /mnt/var

mount /dev/$(DRIVENAME)/var /mnt/var

### /var/log and /var/log/audit

mkfs.ext4 /dev/$(DRIVENAME)/log

mkdir /mnt/var/log

mount /dev/$(DRIVENAME)/log /mnt/var/log

mkdir /mnt/var/log/audit

mount /dev/$(DRIVENAME)/log /mnt/var/log/audit

### /tmp and /var/tmp

mkfs.ext4 /dev/$(DRIVENAME)/tmp

mkdir /mnt/tmp

mount /dev/$(DRIVENAME)/tmp /mnt/tmp

mkdir /mnt/var/tmp

mount /dev/$(DRIVENAME)/tmp /mnt/var/tmp

### /usr

mkfs.ext4 /dev/$(DRIVENAME)/usr

mkdir /mnt/usr

mount /dev/$(DRIVENAME)/usr /mnt/usr

### /home

mkfs.ext4 /dev/$(DRIVENAME)/home

mkdir /mnt/home

mount /dev/$(DRIVENAME)/home /mnt/home

### generate fstab

mkdir /mnt/etc
genfstab -U -p /mnt >> /mnt/etc/fstab

# INSTALL

pacstrap -i /mnt base base-devel linux-firmware linux-hardened linux  
⏎

arch-chroot /mnt

pacman -S linux-headers linux-hardened-headers
⏎

pacman -S nano 
⏎

## networking 

pacman -S openssh networkmanager wpa_supplicant wireless_tools dialog openvpn networkmanager-openvpn networkmanager-vpnc network-manager-applet dhclient libsecret nm-connection-editor modemmanager usb_modeswitch rp-ppoe networkmanager-openconnect networkmanager-pptp networkmanager-l2tp 
⏎

systemctl enable NetworkManager


## lvm support

pacman -S lvm2
⏎

nano /etc/mkinitcpio.conf 

    HOOKS....block encrypt lvm2 filesystem

mkinitcpio -p linux

mkinitcpio -p linux-hardened

## generate locales

nano /etc/locale.gen
    en_us.UTF-8 UTF-8
    en_us ISO-8859-1

locale-gen

## Setup Root password and User

passwd
>enter root password

useradd -m -g users -G wheel $(USER) <!--- Setup User -->

passwd $(USER)
>enter user password

EDITOR=nano visudo
    %wheel ALL=(ALL) ALL

# BOOTLOADER
pacman -S grub efibootmgr dosfstools os-prober mtools
⏎

mkdir /boot/EFI

mount /dev/sda1 /boot/EFI

grub-install --target=x86_64-efi --bootloader-id=grub_uefi --recheck

cp /usr/share/locale/en\@quot/LC_MESSAGES/grub.mo /boot/grub/locale.mo

nano /etc/default/grub

    GRUB_CMDLINE_LINUX_DEFAULT="cryptdevice=/dev/sda3:$(DRIVENAME):allow-discards loglevel=3 quiet"


    GRUB_ENABLE_CRYPTODISK=y

pacman -S intel-ucode mesa

grub-mkconfig -o /boot/grub/grub.cfg

# Secure Boot 
replacement for grub


pacman -S systemd efitools

bootctl status

#### backup old boot

efi-readvar -v PK -o old_PK.esl

efi-readvar -v KEK -o old_KEK.esl

efi-readvar -v db -o old_db.esl

efi-readvar -v dbx -o old_dbx.esl

#### Gen keys

uuidgen --random > GUID.txt

 openssl req -newkey rsa:4096 -nodes -keyout PK.key -new -x509 -sha256 -days 3650 -subj "/CN=my Platform Key/" -out PK.crt

 openssl x509 -outform DER -in PK.crt -out PK.cer

 cert-to-efi-sig-list -g "$(< GUID.txt)" PK.crt PK.esl

 sign-efi-sig-list -g "$(< GUID.txt)" -k PK.key -c PK.crt PK PK.esl PK.auth


 sign-efi-sig-list -g "$(< GUID.txt)" -c PK.crt -k PK.key PK /dev/null rm_PK.auth

 openssl req -newkey rsa:4096 -nodes -keyout db.key -new -x509 -sha256 -days 3650 -subj "/CN=my Signature Database key/" -out db.crt

 openssl x509 -outform DER -in db.crt -out db.cer

 cert-to-efi-sig-list -g "$(< GUID.txt)" db.crt db.esl

 sign-efi-sig-list -g "$(< GUID.txt)" -k KEK.key -c KEK.crt db db.esl db.auth

 mkdir /etc/efi-keys

 cd !$

 curl -L -O https://www.rodsbooks.com/efi-bootloaders/mkkeys.sh

 chmod +x mkkeys.sh

 ./mkkeys.sh

<Enter a Common Name to embed in the keys, e.g. "Secure Boot">

### updating keys to KEK
cert-to-efi-sig-list -g "$(< GUID.txt)" new_db.crt new_db.esl

sign-efi-sig-list -g "$(< GUID.txt)" -k KEK.key -c KEK.crt db new_db.esl new_db.auth

Copy /usr/share/libalpm/hooks/90-mkinitcpio-install.hook to /etc/pacman.d/hooks/90-mkinitcpio-install.hook and /usr/share/libalpm/scripts/mkinitcpio-install to /usr/local/share/libalpm/scripts/mkinitcpio-install.

In /etc/pacman.d/hooks/90-mkinitcpio-install.hook, replace: 

Exec = /usr/share/libalpm/scripts/mkinitcpio-install

with:

Exec = /usr/local/share/libalpm/scripts/mkinitcpio-install

In /usr/local/share/libalpm/scripts/mkinitcpio-install, replace:

install -Dm644 "${line}" "/boot/vmlinuz-${pkgbase}"

with:

sbsign --key /path/to/db.key --cert /path/to/db.crt --output "/boot/vmlinuz-${pkgbase}" "${line}"

#### fully automated efi signing
https://github.com/andreyv/sbupdate 

https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface/Secure_Boot

<!--- once working remove grub --->
https://github.com/andreyv/sbupdate

## SWAP
dd if =/dev/zero of=/swapfile bs=1M count=64000 <!--- AMOUNT OF RAM --> status=progress

chmod 600 /swapfile

mkswap /swapfile

echo '/swapfile none swap sw 0 0' | tee -a /etc/fstab

swapon -a

## Timezone 

timedatectl set-timezone America/Denver

systemctl enable systemd-timesyncd

## Hostname

hostnamectl set-hostname $(HOSTNAME) <!--- hostname-->


nano /etc/hosts

    127.0.0.1 localhost
    127.0.1.1 $(HOSTNAME)

# Firewall
pacman -S iptables 

# REBOOT
exit

umount -a

reboot <!--- Remove Bootable Drive -->


## X11

pacman -S xorg-server xorg-xinit

i3 vs dwm??

## Wayland and Sway
replacemnt for xorg and i3

## Bash

nano .bashrc 

    PS1='\[[\e[01;31m\]\u\[\e[33m\]@\[\e[35m\]\h\[\e[00m\]]:\[\e[01;34m\]\w\[\e>

https://raw.githubusercontent.com/ChrisTitusTech/scripts/master/fancy-bash-promt.sh

## Code
pacman -S git curl
 code rust python python-pip jre-openjdk-headless jre-openjdk jdk-openjdk

## Yaourt 
cd /opt

sudo git clone https://aur.archlinux.org/yay.git

    <!--- to find user group 
     id $(USER) -->

sudo chown -R $(USER):wheel ./yay
cd yay

makepkg -si

## Tools
firefox ranger

## Communication 
irssi

## Theme
i3-gaps polybar rofi feh nerd-fonts-complete 

## Fonts
powerline-fonts
