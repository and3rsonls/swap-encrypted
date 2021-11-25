swap encryption

###without suspend-to-disk
# vim /etc/crypttab
	--->SET
	---># <name>  <device>     <password>     <options>
	--->swap      /dev/sdX#    /dev/urandom   swap,cipher=aes-cbc-essiv:sha256,size=256

# vim /etc/fstab
	--->SET
	--->/dev/mapper/swap  none   swap    defaults   0 0
------------------------------------

###---used programs

---cryptsetup (LUKS)
---grub (bootloader)
---mkinitcpio (initial ramdisk)
---mkinitcpio-openswap (create a hook file)

###with suspend-to-disk
# pacman -Syu --needed cryptsetup
$ yay -Syu mkinitcpio-openswap
	--->install programs

###---preparations
# swapoff /dev/sdX1
	--->umount swap device

# mkdir -p /etc/initcpio/keys
	--->make a dir

# dd if=/dev/random of=/etc/initcpio/keys/openswapkey.key bs=512 count=8
	--->make a key


###---LUKS
# cryptsetup luksFormat /dev/sdX1
	--->create luks container

# cryptsetup open /dev/sdX1 cryptswap
	--->open container

# mkswap /dev/mapper/cryptswap
	--->make swap

# chmod 0600 /etc/initcpio/keys/*.key
# chattr +i /etc/initcpio/keys/*.key
	--->set proper permissions and make it real hard to accidentally do something to these files

# cryptsetup luksAddKey /dev/sdX1 /etc/initcpio/keys/openswapkey.key
	--->add cryptkey

# cryptsetup luksClose /dev/mapper/cryptswap
	--->close device

# cryptsetup open --key-file /etc/initcpio/keys/openswapkey.key /dev/sdX1
	--->open device with a key

###---persistent block device naming
# ls -l /dev/disk/by-id
	--->schemes for persistent naming by-uuid

###---recreate initial ramdisk
# vim /etc/mkinitcpio.conf
	--->SET
	--->HOOKS=(consolefont base udev openswap encrypt resume lvm2 keyboard keymap autodetect modconf block fsck filesystems)

# mkinitcpio -p linux-lts
	--->run

###---recreate bootloader with GRUB
# vim /etc/default/grub
	--->SET
	--->GRUB_CMDLINE_LINUX_DEFAULT="kernel /vmlinuz-linux cryptdevice=/dev/disk/by-uuid/XXXXXXXX-1111-1111-1111-XXXXXXXXXXXX(/dev/sdX1):cryptswap root=/dev/disk/by-uuid/XXXXXXXX-3333-3333-3333-XXXXXXXXXXXX(/dev/sdX3) resume=/dev/disk/by-uuid/XXXXXXXX-!!!!-!!!!-!!!!-XXXXXXXXXXXX(/dev/mapper/cryptswap) ro loglevel=3"

# grub-mkconfig -o /boot/grub/grub.cfg
	--->run

###---setup openswap
# vim /etc/initcpio/hooks/openswap
	--->SET hook
---
run_hook ()
{
mkdir key_device
mount /dev/disk/by-uuid/XXXXXXXX-3333-3333-3333-XXXXXXXXXXXX key_device
cryptsetup open --key-file key_device/etc/initcpio/keys/openswapkey.key /dev/disk/by-uuid/XXXXXXXX-1111-1111-1111-XXXXXXXXXXXX cryptswap
umount key_device
}
---

# vim /etc/openswap.conf
	--->SET
---
#/dev/mapper/cryptswap
swap_device=/dev/disk/by-uuid/XXXXXXXX-!!!!-!!!!-!!!!-XXXXXXXXXXXX
crypt_swap_name=cryptswap

#/dev/sdX3
keyfile_device=/dev/disk/by-uuid/XXXXXXXX-3333-3333-3333-XXXXXXXXXXXX
keyfile_filename=etc/initcpio/keys/openswapkey.key

cryptsetup_options="--type luks"
---

###---setup fstab
# vim /etc/fstab
	--->SET
# <file system>                             <dir>       <type>      <options>   <dump> <pass>
# /dev/mapper/cryptswap         CRYPT
UUID=0b97da15-68cd-4462-be39-e5eebcb7091a   none        swap        defaults    0       0

###---sources
https://wiki.archlinux.org/title/Dm-crypt/Swap_encryption
https://wiki.archlinux.org/title/Persistent_block_device_naming#by-id_and_by-path
https://wiki.archlinux.org/index.php?title=Talk:Dm-crypt&oldid=255742#Suspend_to_disk_instructions_are_insecure
https://blog.hackeriet.no/lvm-in-luks-with-encrypted-boot-partition-and-suspend-to-disk
