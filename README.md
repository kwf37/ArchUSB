# ArchUSB
https://wiki.archlinux.org/index.php/Installing_Arch_Linux_on_a_USB_key
https://wiki.archlinux.org/index.php/Install_from_existing_Linux

## Steps from linux
* First partition the disk
* Make a temporary directory /tmp
* Download boostrap image with wget into tmp, with a command like
`wget mirrors.rit.edu/archlinux/iso/2018.07.01/archlinux-bootstrap-2018.07.01-x86_64.tar.gz`.*
* Extract using `tar -xzf <filename>`
* cd /tmp/root.x86_64
* Prepare the chroot and chroot to root.x86_64
* run `pacman-key --init` and `pacman-key --populate-archlinux`
* enable a mirror by editing /tmp/root.x86_64/etc/pacman.d/mirrorlist and uncommenting a mirror
* refresh packages usinh pacman -Syyu
* install packages you intend to use
* Make filesystems and mount them

## Partition Scheme
https://wiki.gentoo.org/wiki/Handbook:AMD64/Installation/Disks

## USB-specific notes
https://wiki.archlinux.org/index.php/Installing_Arch_Linux_on_a_USB_key


## Packages to Install
* vi
* i3
* xserver
* firefox
* networkmanager
* wpa_supplicant

## wpa_supplicant

start with `wpa_supplicant -B -i interface -c /etc/wpa_supplicant/wpa_supplicant.conf`
run `wpa_cli`

