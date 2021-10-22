# Hardening

# DISK SETUP

fdisk -l

fdisk /dev/sda <!--- or users drive -->

## Hardening Checks

g <!--- new partition table -->

n
⏎ 
1
⏎ 
+200M <!--- for efi -->
⏎ 
t
⏎ 
1

n
⏎ 
2
⏎
⏎ <!--- Rest of Storage -->
t
⏎ 
30 <!--- for LVM -->
⏎ 
w

## 2 partions time for formating
mkfs.fat -F32 /dev/sda1 

mkfs.ext4 /dev/sda2

## time to encrypt
cryptsetup luksFormat /dev/sda2 

YES
> enter encryption passphrase

cryptsetup open --type luks /dev/sda2 $(DRIVE) 
> enter encryption passphrase

## setup lvm, 'physical' volume, and logical volumes

pvcreate --dataalignment 1m /dev/mapper/$(DRIVE)

vgcreate $(DRIVENAME) /dev/mapper/$(DRIVE) <!--- Volume Name -->

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

mkdir /mnt/efi

mount /dev/sda1 /mnt/efi

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


GOTO INSTALLATION.MD
