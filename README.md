# ArchUSB

## Steps from linux
* Make a temporary directory /tmp
* Download boostrap image with wget into tmp, with a command like
`wget mirrors.rit.edu/archlinux/iso/2018.07.01/archlinux-bootstrap-2018.07.01-x86_64.tar.gz`.
Extract using `tar -xzf <filename>`

## Partition Scheme

## USB-specific notes
https://wiki.archlinux.org/index.php/Installing_Arch_Linux_on_a_USB_key


## Packages to Install

## wpa_supplicant

start with `wpa_supplicant -B -i interface -c /etc/wpa_supplicant/wpa_supplicant.conf`
run `wpa_cli`

