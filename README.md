# The Build
I'll be writing up my experience building a home fileserver for streaming media throughout my house, backing up my MacBook to it with Time Machine, and just generally storing lots and lots of stuff. When it's complete, it will have 18TB of usable storage, in a ZFS raidz2 pool. But, before all the drives are put into raidz2, I want to play with them and run some different configurations with them.

## Hardware

* Case: Cooler Master Silencio 652S
* PSU: be quit! Straight Power 10 500W Modular
* Motherboard: Supermicro X10SL7-F
* CPU: Intel Core i3-4360 @ 3.7GHz
* RAM: Crucial 8GB (4GBx2) DDR3 1600 Unbuffered ECC
* OS Drive: Mushkin Enhanced Chronos 2.5" 60GB SATA3 SSD 
* Storage Drives: 8x 3TB Western Digital Red NAS Hard Drives

Putting everything together was pretty straightforward. The case actually has some really cool features. The main hard drive cage can convert from 2.5" to 3.5" by removing some screws and setting it out further. The main cage holds 4 drives, and the secondary cage 3 drives. The case also includes a plastic mounting tray to mount a 3.5" drive a 5.25" bay. There's also three places you can mount a 2.5" SSD within the case; behind the motherboard, the bottom of the case, and the bottom of the 5.25" cage. I chose to mount my SSD on the bottom of the case, due to the lenth of my SATA connectors. The case allows for some pretty good wire management/hiding, but with 9 drives and soundproofing material on the case panels it gets really tight in there. Was quite difficult to get the back panel closed.

## Pictures
TODO

## Initial Boot
Once I got everything together, I grabbed a monitor and my Poker II keyboard excited to boot for the first time. Plug everything in, plug the PSU into the wall, hit the power button... Nothing. Lights on the motherboard are on, but nothing happens. No fans spinning, not even the CPU fan. After a few hours of troubleshooting with the help of a good friend of mine and some helpful strangers on IRC, I had determined it was a power problem (next time I should RTFM a little better) and ripped the motherboard out and connected only power to the board and CPU and lo-and-behold it turned on. As I was putting it back together, I realized I had one more standoff installed in the case that was causing the motherboard to short. Goddamnit.

# Home Networking
Before I could go any further... I had to finish setting up the network in my house.

When I did the restoration of my house, I ran all Cat6 wires, but never terminated any of them and set anything up. So I also ordered a 12-port patch panel, and ran to Home Depot to pickup crimpers and connectors and what-not. All of the ethernet in the house was run so that it could all be terminated in the closet of my new addition on the house. The moden is in the living room, since that's where Verizon installed it, so a single cable runs from there to the closet in the back of the house. The upstairs bedrooms run down and back into the closet. The addition also has 2 cables running to the closet.

After wiring the 6 cables into the patch panel, it was time to determine which port was the WAN. The only thing I had in the house that had an ethernet port (aside from the now 100 pound file server being built) was a Raspberry Pi with Arch already installed on it and setup to use a static IP. So, I plugged it into each port one at a time and pinged it from my MacBook. Next I had to determine which port on the patch panel went to the left upstairs bedroom, where the fileserver will reside. *port switching intensifies*

# Installing Arch
Finally, time to install Arch on the server.

I will be following through the Beginners' Guide from the spectacular Arch Wiki. I won't be copy and pasting it all into this writeup, only commenting on sections where I had different input than the default. This includes the contents of my configuration files, at the time of this writeup.

## Prerequisites
I'm using a Supermicro X10SL7-F motherboard, which supports UEFI so we're using UEFI.
I created a USB stick installer for Arch.
The first thing I had to do was update the boot order to boot the USB stick in UEFI boot mode.

## Prepare the storage devices
Arch was being installed on /dev/sda, a Mushkin 60GB SSD.

I partitioned using `parted`. I'm using `xfs` for `/`, `/var`, and `/home`. `/boot` is `fat32 (vfat)`.

```
# parted /dev/sda
(parted) mklabel gpt
(parted) mkpart ESP fat32 1M 513M
(parted) set 1 boot on
(parted) name 1 boot
(parted) mkpart primary xfs 513M 20.5G
(parted) name 2 root
(parted) mkpart primary xfs 20.5G 32.5G
(parted) name 3 var
(parted) mkpart primary xfs 32.5G 100%
(parted) name 4 home
(parted) quit
# mkfs.vfat /dev/sda1
# mkfs.xfs /dev/sda{2,3,4}
```

## Select a mirror
Generated one with the Mirrorlist Generator, specificly selecting US mirrors only.

## Time zone
`# ln -s /usr/share/zoneinfo/America/New_York /etc/localtime`

## Hostname
I chose to start using GoT character names for my computer names, started with this file server. I decided to go with `littlefinger` for the hostname after **Petyr Baelish**, as he collects intelligence and data.

## Configure the network
I'm going with a static IP for my box, `192.168.1.200`.

```
# cat | sudo tee /etc/netctl/local_network >/dev/null
Description='A basic static ethernet connection'
Interface=eno1
Connection=ethernet
IP=static
Address=('192.168.1.200/24')
Gateway='192.168.1.1'
DNS=('8.8.8.8' '8.8.4.4')
<C-d>
# netctl enable local_network
```

## Install and configure a bootloader
We're using UEFI, so here are the commands I had to use.

```
# pacman -S dosfstools efibootmgr gummiboot
# gummiboot --path=/boot install
# cat | sudo tee /boot/loader/entries/arch.conf >/dev/null
title	Arch Linux
linux	/vmlinuz-linux
initrd	/initramfs-linux.img
options	root=/dev/sda2 rw
<C-d>
# cat | sudo tee /boot/loader/loader.conf >/dev/null
default arch
timeout 5
<C-d>
```

## Post-installation

### Install vim
`# pacman -Syy vim`

### Install openssh
```
# pacman -Syy openssh
# systemctl start sshd.service
# systemctl enable ssh.service
```

### Add my user
```
# useradd -m -G wheel -s /bin/bash marc
# passwd marc
```

### Install yaourt
```
$ mkdir builds && cd $_
$ curl -L -O https://aur.archlinux.org/packages/pa/package-query/package-query.tar.gz
$ cd package-query && makepkg -s && cd ..
$ curl -L -O https://aur.archlinux.org/packages/ya/yaourt/yaourt.tar.gz
$ cd yaourt && makepkg -s && cd ..
```

### Install archey3
```
$ yaourt archey3-git
$ cat > ~/.archey3.cfg
[core]
display_modules = distro(), uname(n), uname(r), uptime(), packages(), ram(), cpu(), fs(/)
align = top
color = blue
<C-d>
$ echo "archey3" >> ~/.bashrc
```

TODO: Insert login picture w/ archey
