## Swap Device Encryption With Hibernation

### Distro
in this case the Archlinux distro is installed and configured.

|  staus now |
|  :---:     |
| devide swap **is not encrypted** and **hibernating** |
| device /boot **is not encrypted** |
| device / **is not encrypted** |

|status after|
| :---:      |
| device swap **encrypted swap** and **hibernating** |
| device /boot **is not encrypted** |
| device / **is not encrypted** |

### Used programs
- cryptsetup (LUKS)
- grub (bootloader)
- mkinitcpio (initial ramdisk)
- AUR mkinitcpio-openswap (create a hook file)

### Preparations
`# swapoff /dev/sdX1`  
umount swap device

`# mkdir -p /etc/initcpio/keys`  
make a dir

`# dd if=/dev/random of=/etc/initcpio/keys/openswapkey.key bs=512 count=8`  
make a key

`# chmod 0600 /etc/initcpio/keys/*.key`  
`# chattr +i /etc/initcpio/keys/*.key`  
proper permissions and make it real hard to accidentally do something to these files

### LUKS
`# cryptsetup luksFormat /dev/sdX1`  
create luks container

`# cryptsetup open /dev/sdX1 cryptswap`  
open container

`# mkswap /dev/mapper/cryptswap`  
make swap

`# cryptsetup luksAddKey /dev/sdX1 /etc/initcpio/keys/openswapkey.key`  
add cryptkey

`# cryptsetup luksClose /dev/mapper/cryptswap`  
close device

`# cryptsetup open --key-file /etc/initcpio/keys/openswapkey.key /dev/sdX1`  
open device with a key

### Persistent block device naming
`# ls -l /dev/disk/by-id`  
schemes for persistent naming by-uuid

### Recreate initial ramdisk
`# vim /etc/mkinitcpio.conf`  
set hooks to  

~~~bash
HOOKS=(consolefont base udev openswap encrypt resume lvm2 keyboard keymap autodetect modconf block fsck filesystems)
~~~

`# mkinitcpio -p linux-lts`  
apply change

### Recreate bootloader with GRUB
`# vim /etc/default/grub`  
set grub to  

~~~bash
GRUB_CMDLINE_LINUX_DEFAULT="cryptdevice=/dev/disk/by-uuid/XXXXXXXX-1111-1111-1111-XXXXXXXXXXXX:cryptswap root=/dev/disk/by-uuid/XXXXXXXX-3333-3333-3333-XXXXXXXXXXXX resume=/dev/disk/by-uuid/XXXXXXXX-!!!!-!!!!-!!!!-XXXXXXXXXXXX ro loglevel=3"
~~~

`# grub-mkconfig -o /boot/grub/grub.cfg`  
to apply change

### Setup openswap
`# vim /etc/initcpio/hooks/openswap`  
edit hook

~~~bash
run_hook ()
{
mkdir key_device
mount /dev/disk/by-uuid/XXXXXXXX-3333-3333-3333-XXXXXXXXXXXX key_device
cryptsetup open --key-file key_device/etc/initcpio/keys/openswapkey.key /dev/disk/by-uuid/XXXXXXXX-1111-1111-1111-XXXXXXXXXXXX cryptswap
umount key_device
}
~~~

`# vim /etc/openswap.conf`  
edit

~~~bash
#/dev/mapper/cryptswap
swap_device=/dev/disk/by-uuid/XXXXXXXX-!!!!-!!!!-!!!!-XXXXXXXXXXXX
crypt_swap_name=cryptswap

#/dev/sdX3
keyfile_device=/dev/disk/by-uuid/XXXXXXXX-3333-3333-3333-XXXXXXXXXXXX
keyfile_filename=etc/initcpio/keys/openswapkey.key

cryptsetup_options="--type luks"
~~~

### Setup fstab
`# vim /etc/fstab`  
edit and add

~~~bash
# <file system>                             <dir>       <type>      <options>   <dump> <pass>
# /dev/mapper/cryptswap         CRYPT
UUID=0b97da15-68cd-4462-be39-e5eebcb7091a   none        swap        defaults    0       0
~~~

### Sources
https://wiki.archlinux.org/title/Dm-crypt/Swap_encryption<br>
https://wiki.archlinux.org/title/Persistent_block_device_naming#by-id_and_by-path<br>
https://wiki.archlinux.org/index.php?title=Talk:Dm-crypt&oldid=255742#Suspend_to_disk_instructions_are_insecure<br>
https://blog.hackeriet.no/lvm-in-luks-with-encrypted-boot-partition-and-suspend-to-disk<br>
