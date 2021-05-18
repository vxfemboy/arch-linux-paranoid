# INTO THE MATRIX

##  Verify
Verify the Arch ISO you got from [here](https://archlinux.org/download/)
using sha1sum:
```
curl -s https://archlinux.org/iso/latest/sha1sums.txt | sha1sum -c
```
All is OK? Lets move forward...

## Live USB
The simplest way to install arch linux is through a usb   
you can create a live usb from a linux enviroment using:
```
dd bs=4M if=/path/to/archlinux.iso of=/dev/sdX status=progress && sync
```
Be sure to replace path/to/archlinux.iso with the iso you verified from the step above.   
Be also sure to replace ```/dev/sdX``` with the installation drive you wish to boot from.

## Boot and Internet
Make sure secure boot is disabled in BIOS and Boot from UEFI on the installation drive.

### Connecting Internet 
You can connect your computer to the internet by plugging an ethernet cable or by using the internet sharing feature on your phone or find ways to connect to the internet using the [Arch Linux Wiki](https://wiki.archlinux.org/index.php/installation_guide#Connect_to_the_internet).
## Wipe Disk 
To see what drives you have you can do:
```
fdisk -l
```   
NOTE: Be sure to replace ```/dev/sdX``` with disc you want to install too.   
You might want to remember this due to us using this throughout this guide.
```
blkdiscard /dev/sdX
shred
sync
```
This will wipe your disks so its more difficult to analyze

## Partitioning
For this I like too have 2 partitions 

Number |Size    |Name         |
-------|--------|-------------|
   1   | +200M  | EFI         |
   2   | FREE  | LVM  |

And find the drive you want to install on then do:
```
fdisk /dev/sdX
```
Now do:
```
g⏎
n⏎
⏎
+200M⏎
t⏎
1⏎
```
This will be our EFI partition.
Now do:
```
n⏎
⏎
⏎
t⏎
30⏎
w⏎
```
This will be our lvm encrypted drive.   
Now we have our partition Table written.

## Formatting 
```
mkfs.fat -F32 /dev/sdX1
mkfs.ext4 /dev/sdX2
```

## Encrypting 
```
cryptsetup luksFormat -c aes-xts-plain64 -h sha512 -S 512 -i 30000 --type luks2 --pbkdf argon2id /dev/sdX2
```
This will setup encryption on our LVM partion, to continue type ```YES``` following with your desired encryption passphrase.
```
cryptsetup open --type luks2 /dev/sdX2 $(DRIVE)
```
NOTE: Replace ```$(DRIVE)``` with your desired drivename.   
Enter your encryption passphrase and return.

## LVM partition
We will make the LVM and volume groups on top of the LUKS2 partition, To do so do:
```
pvcreate --dataalignment 1m /dev/mapper/$(DRIVE)
```
Now we must create the volume group:
```
vgcreate $(VOLUME) /dev/mapper/$(DRIVE)
```
NOTE: Make sure to replace ```$(VOLUME)``` with your desired volume name.
```
lvcreate -L 100GB $(VOLUME) -n root
lvcreate -L 10GB $(VOLUME) -n var
lvcreate -L 5GB $(VOLUME) -n log
lvcreate -L 5GB $(VOLUME) -n tmp
lvcreate -L 50GB $(VOLUME) -n usr
lvcreate -l 100%FREE $(VOLUME) -n home
```
NOTE: Feel free to change the storage options as you desire   
You can see why we make a seperate partition for each [here](https://github.com/trimstray/linux-hardening-checklist)

### Activate Volumes
```
modprob dm_mod
vgscan
vgchange -ay
```

## Format and Mount
#### For /root
```
mkfs.ext4 /dev/$(DRIVENAME)/root
mount /dev/$(DRIVENAME)/root /mnt
```
Now to make all the directories:
```
mkdir /mnt/{efi,dev,var,var/log,var/log/audit,tmp,var/tmp,usr,home,etc}
```
#### For /boot
```
mount /dev/sda1 /mnt/efi
```
#### For /var
```
mkfs.ext4 /dev/$(DRIVENAME)/var
mount /dev/$(DRIVENAME)/var /mnt/var
```
#### For /var/log and /var/log/audit
```
mkfs.ext4 /dev/$(DRIVENAME)/log
mount /dev/$(DRIVENAME)/log /mnt/var/log
mount /dev/$(DRIVENAME)/log /mnt/var/log/audit
```
#### For /tmp and /var/tmp
```
mkfs.ext4 /dev/$(DRIVENAME)/tmp
mount /dev/$(DRIVENAME)/tmp /mnt/tmp
mount /dev/$(DRIVENAME)/tmp /mnt/var/tmp
```
#### For /usr
```
mkfs.ext4 /dev/$(DRIVENAME)/usr
mount /dev/$(DRIVENAME)/usr /mnt/usr
```
#### For /home
```
mkfs.ext4 /dev/$(DRIVENAME)/home
mkdir /mnt/home
mount /dev/$(DRIVENAME)/home /mnt/home
```

## Generate FSTAB
```
genfstab -U -p /mnt >> /mnt/etc/fstab
```

## Pre-Chroot
```
pacstrap -i /mnt base base-devel linux-firmware linux-hardened linux-hardened-headers man-db man-pages lvm2 nano mkinitcpio sudo efibootmgr efitools git wget curl texinfo intel-ucode 
```
NOTE: you can also change ```intel-ucode``` a compatible microcode for your system like ```amd-ucode``` if you have AMD

## Chroot
```
arch-chroot /mnt
```

### Timezone 
Change your timezone:
```
timedatectl set-timezone Region/City
ln -sf /usr/share/zoneinfo/Region/City /etc/localtime
hwclock --systohc
systemctl enable systemd-timesyncd
```

Type ```nano /etc/locale.gen``` and uncomment:
````
en_us.UTF-8 UTF-8
en_us ISO-8859-1
````
```locale-gen```
```
echo "LANG=en_US.UTF-8" > /etc/locale.conf
````
NOTE: Feel Free to change the locales as nessacary 

### Hostname and Hosts
```
hostname=$(HOSTNAME)
```
change ```$(HOSTNAME)``` to your desired hostname
```
echo $hostname > /etc/hostname

echo -e "127.0.0.1 localhost\n::1 localhost\n127.0.1.1 $hostname.localdomain $hostname" > /etc/hosts
```

### Edit MKINITCPIO
```nano /etc/mkinitcpio.conf``` 
```
MODULES=(crc32c-intel)

HOOKS=(base udev autodetect keyboard keymap modconf block encrypt lvm2 filesystems fsck)
```
Note: obviously, use the ```crc32c-intel``` module if you are using an Intel CPU that support Intel SSE4.2, otherwise, use the ```crc32c``` module.

Note2: if you are only using the QWERTY keyboard layout, you can not include the keymap hook.
Enableing zstd compression for initramfs
```
sed -i 's/#COMPRESSION="zstd"/COMPRESSION="zstd"/g' /etc/mkinitcpio.conf
```
```mkinitcpio -p linux-hardened```
## Add keylayout:
```
echo "KEYMAP=en" > /etc/vconsole.conf
````

## Setup root and new user
```passwd```
enter root password

```useradd -m -g users -G wheel $(USER)```
NOTE: Replace ```$(USER)``` with your desired username
```passwd $(USER)```
enter user password

```EDITOR=nano visudo ``` and uncomment:
```
%wheel ALL=(ALL) ALL
```

### install gpu drivers
```pacman -S mesa vulkan-intel```
NOTE: feel free to replace this with what you need ```nvidia virtualbox-guest-utils xf86-video-vmware xf86-video-ati```

### Preventing Core Dumps 
```
echo "Storage=none" >> /etc/systemd/coredump.conf
```

### Limiting JournelID 
```sed -i 's/#SystemMaxUse=/SystemMaxUse=250M/g' /etc/systemd/journald.conf```

### Allowing Force reboot on CTRL+ALT+DELTE*7
```
sed -i 's/#CtrlAltDelBurstAction=reboot-force/CtrlAltDelBurstAction=reboot-force/g' /etc/systemd/system.conf
```

### Reducing shutdown timeout 
```
sed -i 's/#DefaultTimeoutStopSec=90s/DefaultTimeoutStopSec=10s/g' /etc/systemd/system.conf
```

### Setup yay AUR
```su $(USER)``` enter your user password

```
cd 
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
cd ..
rm -rf yay
yay --sudoloop --save
yay -S sbupdate-git sbkeys
exit
```


## Updating sbupdate.conf
```
sed -i 's/#CMDLINE_DEFAULT=""/CMDLINE_DEFAULT="cryptdevice=UUID='"$(blkid -s UUID -o value /dev/sdX2)"':$(DRIVE) root=\/dev\/\/root rootflags=subvol=@ rw loglevel=3 rd.udev.log_priority=3 rd.systemd.show_status=false systemd.show_status=false quiet splash apparmor=1 lsm=lockdown,yama,apparmor i915.fastboot=1"/g' /etc/sbupdate.conf
```
NOTE: Remove the i915.fastboot=1 option if you are not using an integrated GPU of an Intel CPU.
```
sed -i 's/#ESP_DIR="\/boot"/ESP_DIR="\/efi"/g' /etc/sbupdate.conf
```
### setup keys
```
pacman -S efitools efivar sbsigntools python python-pip python2 python2-pip efibootmgr openssl
mkdir /etc/efi-keys
cd !$
wget https://www.rodsbooks.com/efi-bootloaders/mkkeys.sh
chmod +x mkkeys.sh
./mkkeys.sh
```

## setup secure boot
```
sbsign --key /etc/efi-keys/DB.key --cert /etc/efi-keys/DB.crt --output /efi/BOOT/Arch/linux-hardened-signed.efi /efi/BOOT/Arch/linux-hardened-signed.efi
```
Sign kernel:
```
sbsign --key /etc/efi-keys/DB.key --cert /etc/efi-keys/DB.crt --output /efi/BOOT/Arch/linux-hardened-signed.efi /efi/BOOT/Arch/linux-hardened-signed.efi
```
boot from kernel:
```
efibootmgr -c -d /dev/sdX -p 1 --label "BunnyArch" -l "BOOT\Arch\linux-hardened-signed.efi" --verbose
```
check if valid boot: 
```efibootmgr```   
verify BunnyArch is first, if not reorder by:   
```efibootmgr -o 0002,2001,2020```   
with the corresponding boot numbers of your list.

# Install 
```
pacman -S openssh networkmanager wpa_supplicant wireless_tools dialog openvpn networkmanager-openvpn networkmanager-vpnc network-manager-applet dhclient libsecret nm-connection-editor modemmanager usb_modeswitch rp-ppoe networkmanager-openconnect networkmanager-pptp networkmanager-l2tp iptables usbguard firejail apparmor htop bpytop dnscrypt-proxy syncthing pulseeffects lsp-plugins jre11-openjdk ntfs-3g ldns gvfs-mtp gocryptfs compsize whois openbsd-netcat net-tools ttf-dejavu ttf-liberation ttf-droid ttf-ubuntu-font-family noto-fonts noto-fonts-emoji pulseaudio alsa-utils alsa-plugins
```
```systemctl enable NetworkManager apparmor```

## usbguard and firejail and apparmor
```
usbguard generate-policy > /etc/usbguard/rules.conf
systemctl enable usbguard
apparmor_parser -r /etc/apparmor.d/firejail-default
mkdir /etc/pacman.d/hooks
wget -O /etc/pacman.d/hooks/firejail.hook https://raw.githubusercontent.com/theo546/my-arch-setup/master/firejail.hook
```

## Reboot into bios
FIRST:
```
dmesg | grep -i secure
```

Enroll the keys in this order:

    db
    KEK
    PK

Note: make sure to last enroll your platform key.
Note2: always prefer the .auth certificates over the others certificate type.

Once on the main menu, select Edit Keys then The Allowed Signatures Database (db).
Now select Add New Key then select the drive and the db certificate on the file list.

Note: if you are doing a dual boot with Windows or if you need the Microsoft Corporation UEFI CA 2011 certificate enrolled, add add_MS_db.auth as a db certificate.

Repeat these steps for KEK.

For the platform key, select Replace Key(s) then select your PK certificate just like earlier.

Once done, press ESC to get back on the main menu, if you have done those steps properly, at the top of the screen, it should show Platform is in User Mode.
To finally finish the setup, press Exit to reboot!

From now on, only your new Arch setup (or any EFI executable signed with your keys) will be able to boot on this PC unless you disable Secure Boot.


### REMOVE ALL KEYS ON ESP PARTITION 
### SET BIOS PASSWORD!
backup luks header
```
cryptsetup luksHeaderBackup /dev/sdX2 --header-backup-file luks_header_backup
```

