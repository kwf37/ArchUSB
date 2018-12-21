# ArchUSB

## Installation Process

### Part 0: Setup a Linux Environment
If you do not already have a running linux environment, you can set up a liveUSB with
a distro of your choice to be a temporary working environment for the installation. This
does not have to be Arch Linux. Note that if you do this you need two USBs, one for the liveUSB,
and one to install Arch on.

I picked Ubuntu because the liveUSB setup has easy-to-follow
documentation, and it has all of the tools needed for this installation. To make an Ubuntu
liveUSB, see the [Ubuntu tutorials](https://tutorials.ubuntu.com/) and search for "bootable usb"
tutorials.

Make your liveUSB, boot it up, and login and connect to wifi, and move on to the next part.

### Part 1: Partition the USB Drive
I personally prefer to format my disks first, but this can also be done between Part 2 and Part 3. It is quite easy to make mistakes in this part, so please double check at every step, and I highly recommend reading about partition schemes from several sources.

Once your linux environment is set up, plug in the USB you want to install Arch on and use `dmesg` to figure out what file the USB is mounted to. `dmesg` stands for *driver message* and prints out messages typically produced by device drivers. When you plug in your USB it should add a new message that dmesg will output, including the file representing your USB device. This file should be of the form `/dev/sdX`, where `X` can be any letter. In my case, it was `/dev/sdc`.

Here is an example output of `dmesg`:

**TODO INSERT PICTURE HERE**

The last few messages talk about USB devices, and we can see that the device is in the `/dev/sdc` file. Because everything in the Linux
filesystem is a file or directory, many devices are represented in the `/dev/` folder, which stands for device files. Further reading can be done [here](https://www.tldp.org/LDP/Linux-Filesystem-Hierarchy/html/dev.html).

Now that we have identified where our device is located, we can partition it. This splits the drive into several logical sections that
are independent of each other. In our case, we will need to make some boot partitions to boot it up, and the rest of our filesystem will
be one monolithic partition. I made a GPT (GUID Partition Table).

My partition scheme had three partitions:

| Partition   | Name  | Size          | Filesystem        |
| ----------- |:-----:|:-------------:|:-----------------:|
| Partition 1 | grub  | 2MB           | VFAT              |
| Partition 2 | boot  | 128MB         | ext4 (no journal) |
| Partition 3 | rootfs| Rest of drive | ext4 (no journal) |

These are free to adjust, but when installing  on a USB drive it is wisest to forgo a swap partition. Swap space is used when a computer is out of RAM so it typically undergoes lots of overwriting, which will shorten the lifetime of the USB.


To make the partition scheme, I used the `parted` tool (docs [here](https://www.gnu.org/software/parted/manual/parted.html)). It has
an interactive mode in the terminal that is pretty easy to work with. I also followed the [Gentoo partitioning tutorial](https://wiki.gentoo.org/wiki/Handbook:AMD64/Installation/Disks), which is very well-written. First, run `parted -a optimal /dev/sdX`, where `X` is the letter corresponding to your drive, to start up `parted`. This may require `sudo`. The arguments add optimal alignment to decrease the number of page fetches. Once you are in the interactive mode of `parted`, run the following commands:

1. `mklabel gpt`: This makes a GPT partition table. Make sure you don't have any important data on your USB drive as this will erase it.
2. `print`: Prints out the current table. Take note of what partitions exist, as you will need to remove them.
3. `rm 1`: Remove all partitions. I only had 1 partition, but look at the output of `print` and remove each of the partitions. The argument is the partition number.
4. `print`: Make sure as a sanity check that you have removed all partitions. You should see an empty table.
5. `unit mib`: This sets the units to megabytes while partitioning.
6. `mkpart primary 1 3`: This makes a partition from 1MB to 3MB.
7. `name 1 grub`: This labels the first partition as "grub"
8. `set 1 bios_grub on`: This sets a flag for booting.
9. `mkpart primary 3 131`: Makes the 128MB boot partition
10. `name 2 boot`: This labels the second partition as "boot"
11. `set 2 boot on`: This sets a flag labelling the EFI partition for UEFI booting.
12. `mkpart primary 131 -1`: Makes the last partition that takes up the rest of the USB drive.
13. `name 3 rootfs`: Labels the last partition.
14. `quit`: Exits the program

If you are operating on a liveUSB, you may get warnings that the kernel could not be informed of changes. The changes are still performed if you `Ignore` these warnings, but you should restart as soon as you finish partitioning so that the kernel is aware of the changes on reboot.

Lastly, we need to set a filesystem for each partition. In accordance with UEFI standards, the 2MB grub partition should be formatted as VFAT, which can be done with mkdosfs. The other choices for filesystems are flexible, so I went with ext4 with the option “^has_journal.” This turns off journaling which typically helps in repairing ungraceful dismounts or shutdowns. This is because journaling again provides a significant amount of extra writes, which will lengthen our USB lifetime.

To make the grub partition VFAT, run `mkdosfs /dev/sdX1`. To make the boot partition ext4 with not journaling, run `mkfs.ext4 -O "^has_journal" /dev/sdX2`. Lastly, do the same thing for the rootfs partition: `mkfs.ext4 -O "^has_journal" /dev/sdX2`.

With that, you should be ready to move on to the next part.

### Part 2: Set Up the Chroot Environment (Optional if Already on Arch)
Now we can prepare to install arch. If you are not on arch linux, then you will need to setup a small arch chroot environment. I followed the Arch wiki's "From a host running another Linux distribution" on their [Install from existing Linux page](https://wiki.archlinux.org/index.php/Install_from_existing_Linux). Here are the steps:

1. Download a bootstrap image and move the zipped file to /tmp
2. Extract the image using `tar xzf` on the file you just moved to `/tmp`. This may require `sudo`.
3. Select a repository server by uncommenting one or more entries in `/tmp/root.x86_64/etc/pacman.d/mirrorlist`
4. Run `mount --bind /tmp/root.x86_64 /tmp/root.x86_64`. This may require `sudo`.
5. Run `/tmp/root.x86_64/bin/arch-chroot /tmp/root.x86_64/`. This may require `sudo`.
	The above command is a script that automates a lot of the mounting required to setup a chroot environment, and automatically calls   chroot for you. If for some reason this doesn’t work, please refer to the alternate options in the above "Install from existing Linux" page.
6. Run `pacman-key --init`
7. Run `pacman-key --populate archlinux`
8. Refresh all package lists by running `pacman -Syyu`.
9. Install any useful tools using `pacman -S <package>`. I only installed `vi` as I already partitioned my drives by this point.

The last few pacman commands should finish the setup for our arch boostrap environment. We can now move on to the actual installation.

### Part 3: Install Arch
In this part I simply followed the [Arch wiki installation page](https://wiki.archlinux.org/index.php/Installation_guide#Mount_the_file_systems) from mounting the file systems. Here are the steps:

0. Format the disks as described in Part 1 if you haven't already.
1. Mount the filesystems. For my example, I ran the following commands to mount my filesystem at `/mnt/`:
    ```
    mount /dev/sdc3 /mnt
    mount /dev/sdc2 /mnt/boot
    mkdir /mnt/boot/efi
    mount /dev/sdc1 /mnt/boot/efi
    ```
    
2. Double check that the `/etc/pacman.d/mirrorlist` file is set correctly, and then run `pacstrap /mnt base`. This effectively copies the `base` package group into the new filesystem rooted at `/mnt`, and it can be run with other packages as arguments. For example, if I also wanted to install the `base-devel` packagr group, I could run `pacstrap /mnt base base-devel`. I **highly** recommend installing a network manager here, such as the `networkmanager` package or `wpa_supplicant`. Both of them have good pages on the wiki, and you will not automatically connect to wifi on the new installation.

3. Configure the system by setting up the `fstab` (file system table). This step is critical to get correct, and *very* easy to mess up! This file is super important as it automates the filesystem mounting at bootup. If it’s wrong, your USB won’t boot! Run `genfstab -U /mnt >> /mnt/etc/fstab` to generate a template fstab in `/mnt/etc/fstab`, then modify the `fstab` file to use either UUID, PARTUUID, or PARTLABEL. By default, it will likely use the `/dev/sdX` identifiers, but these are not persistent over reboots. I chose to replace them with partition labels as they are easier to type than the Universally Unique IDentifiers. To determine the UUID, PARTUUID, and PARTLABEL's of the different partitions, run `blkid /dev/sdx*`. This will run the `blkid` on all of your USB partitions, which prints out attributes of a block device, including UUID’s, PARTUUID’s, and PARTLABEL’s, if applicable. **TODO ADD EXAMPLE fstab HERE**

4. Run `chroot /mnt` to change root to the new filesystem.
5. Run `ln -sf /usr/share/zoneinfo/Region/City /etc/localtime` to set the time zone. Replace Region and City with your region (maybe country) and time zone in that region. 
6. Run `hwclock --systohc` to configure the system clock.
7. Uncomment `en_US.UTF-8 UTF-8` and other needed locales in `/etc/locale.gen`, and then run `locale-gen` to generate the locales.
8. Set the `LANG` variable in `/etc/locale.conf`. In my case, I added `LANG=en_US.UTF-8`. You may need to ccreate the file first.
9. Create the hostname file `/etc/hostname`, and enter your hostname there.
10. Create `/etc/hosts` and make it consistent with the hostname file. It should look like the following:
    ```
    127.0.0.1	localhost
    ::1		localhost
    127.0.1.1	myhostname.localdomain	myhostname
    ```
11. Set the root passwd by running `passwd`.
12. Configure journald to use RAM to minimize USB disk access. Modify `/etc/systemd/journald.conf.d` to set the following two variables:
    ```
    Storage=volatile
    RuntimeMaxUse=30M
    ```
    
With that, we are almost done!

### Part 4: Install Bootloader (GRUB2)
The last thing to do is install the bootloader, and then we will have a barebones working arch linux installation on our USB!

1. Download grub2 by running `pacman -S grub`
2. Run `grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=grub --removable --recheck` to install grub on the USB. Make sure to keep the removable option, as we are installing on a USB drive.
3. Run `grub-mkconfig -o /boot/grub/grub.cfg` to generate the grub config. Don't forget this step!
4. Reboot, and hopefully it all works!
5. For troubleshooting, I frequently messed up my partitioning, fstab, and grub boot directory (and forgetting to make a grub config)

If all went well, you should see an arch terminal on bootup! It should not be in emergency mode, and it should not be a grub terminal.

## Post-Installation
After installation, you are free to customize your linux. Here are two super helpful pages from the Arch Wiki:
* https://wiki.archlinux.org/index.php/general_recommendations
* https://wiki.archlinux.org/index.php/list_of_applications

The rest of the section are my notes on the programs I chose and how I customized some of them.

**TODO**
### Connecting to Wifi (Using NetworkManager)
In order to connect to wifi and save configurations, you will need a network manager. Various network managers can be compared [here](https://wiki.archlinux.org/index.php/Network_configuration#Network_managers). I chose to go with NetworkManager, and have documented some basic setup here.

A good first step on the new installation is to configure NetworkManager and connect to wifi. This assumes it was one of the packages installed in Part 3 Step 2.

1. Run `systemctl enable --now NetworkManager.service`. Systemd is the core system/service manager for Arch linux, and its commands can be run with `systemctl`. `systemctl enable --now <service>` adds a service to the services run at bootup, and the `--now` flag also starts it immediately. By calling this on the NetworkManager service, we ensure that the nm (short for NetowrkManager) daemon runs at bootup. The [systemd page](https://wiki.archlinux.org/index.php/systemd) on the Arch wiki is worth keeping handy.
2. Try out the network manager client by running `nmcli device wifi list`. This should list available wifi networks in nice colored font.
3. Try connecting to wifi using `nmcli device wifi connect <SSID> password <password>`.
4. Once successfully connected, your configuration should automatically be saved and your computer will try to connect to it on bootup! Take a dive into the extra features of NetworkManager on [its page on the Arch Wiki](https://wiki.archlinux.org/index.php/NetworkManager). In case you have not noticed already, the Arch wiki is an incredible resource with vast amounts of information on Arch packages and tutorials, so it is worth perusing thoroughly.

### Setting up Xorg and Running I3 and Firefox
Xorg the most common display server, and so in order to run GUI programs in windows, you will need to install Xorg. [This page](https://wiki.archlinux.org/index.php/xorg#Installation) should be your primary reference when working with Xorg and looking for things to optimize and customize.

Since I will be using  i3 window manager for this tutorial, be sure to also read up on [this page](https://wiki.archlinux.org/index.php/i3).

Last important link is the [page on firefox](https://wiki.archlinux.org/index.php/Firefox) on the Arch wiki.

1. Install the whole `xorg` package group. I chose to install the whole package because I have not decided to scrap any of the packages in the group. Read up on [pacman](https://wiki.archlinux.org/index.php/Pacman) if you ever forget how to install packages.
2. Install the whole `xorg-drivers` package group. I chose to install the whole package group because I intend to run the USB on multiple computers.
3. Install the `i3` package group. 
4. Install `firefox`. Default options are fine, but read the wiki page and be sure to pick what you want
5. Install a terminal emulator. [This page](https://wiki.archlinux.org/index.php/List_of_applications#Terminal_emulators) of the Arch wiki has lots of fun choices. I picked `urxvt` and haven't tried many others, so feel free to pick whichever interests you.
6. Install `xorg-xinit`. This will let us start up an X server session from our barebones terminal. Read up on it [here](https://wiki.archlinux.org/index.php/Xinit).
7. Copy the default `.xinitrc` file into your home directory by running `cp /etc/X11/xinit/xinitrc ~/.xinitrc`.
8. Modify the default `.xinitrc` by replacing
    ```
    twm &
    xclock -gemoetry 50x50-1+1 &
    xterm -geometry 80x50+494+51 &
    xterm -geometry 80x20+494-0 &
    exec xterm -geometry 80x66+0+0 -name login
    ```
    with `exec i3`.
9. Call startx which should start up a plain i3 session.
10. The i3 config wizard should pop up in the middle of the screen. Read up on i3 and pick whichever configs suit you.
11. Open up a terminal session, and then in the terminal, open up firefox.
12. See **TODO SCREENSHOT** for what my session looks like. (Temporarily this is just saved as barebones_desktop, where I have a terminal, GIMP, and firefox open).

### Setting Up Sound (TODO)
1. Install `alsa-utils`
2. Run `amixer sset Master unmute`
3. Run `amixer sset Speaker unmute`
4. Run `amixer sset Headphone unmute`

### Power Management (TODO)

### Screen Brightness (TODO)

### Clipboard Manager (TODO)

### Trash/Recycling Manager (TODO)

### File Explorer (TODO)

### System Administration (TODO)

### Customizing i3 (TODO)

### Customizing urxvt (TODO)

### Customizing vim (TODO)


## References
**TODO** reformat the references
* https://wiki.archlinux.org/index.php/Installing_Arch_Linux_on_a_USB_key
* https://wiki.archlinux.org/index.php/Install_from_existing_Linux
* https://wiki.gentoo.org/wiki/Handbook:AMD64/Installation/Disks
* https://www.tldp.org/LDP/Linux-Filesystem-Hierarchy/html/dev.html


