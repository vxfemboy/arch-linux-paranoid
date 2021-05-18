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
dd bs=4M if=/path/to/archlinux.iso of=/dev/sdx status=progress && sync
```
Be sure to replace path/to/archlinux.iso with the iso you verified from the step above

Be also sure to replace /dev/sdx with the installation drive you wish to boot from

## Boot and Internet
Make sure secure boot is disabled in BIOS and Boot from UEFI on the installation drive.

You can connect your computer to the internet by plugging an ethernet cable or by using the internet sharing feature on your phone.  
You can get more ways to connect to the internet using the [Arch Linux Wiki](https://wiki.archlinux.org/index.php/installation_guide#Connect_to_the_internet).
