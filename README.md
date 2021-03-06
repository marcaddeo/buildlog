# The Build
I'll be writing up my experience building a home fileserver for streaming media throughout my house, backing up my MacBook to it with Time Machine, and just generally storing lots and lots of stuff. When it's complete, it will have 16TB of usable storage, in a ZFS raidz2 pool.

## Hardware

* Case: Cooler Master Silencio 652S
* PSU: be quiet! Straight Power 10 500W Modular
* Motherboard: Supermicro X10SL7-F
* CPU: Intel Core i3-4360 @ 3.7GHz
* RAM: Crucial 8GB (4GBx2) DDR3 1600 Unbuffered ECC
* OS Drive: Mushkin Enhanced Chronos 2.5" 60GB SATA3 SSD
* Storage Drives: 8x 3TB Western Digital Red NAS Hard Drives

Putting everything together was pretty straightforward. The case actually has some really cool features. The main hard drive cage can convert from 2.5" to 3.5" by removing some screws and setting it out further. The main cage holds 4 drives, and the secondary cage 3 drives. The case also includes a plastic mounting tray to mount a 3.5" drive a 5.25" bay. There's also three places you can mount a 2.5" SSD within the case; behind the motherboard, the bottom of the case, and the bottom of the 5.25" cage. I chose to mount my SSD on the bottom of the case, due to the lenth of my SATA connectors. The case allows for some pretty good wire management/hiding, but with 9 drives and soundproofing material on the case panels it gets really tight in there. Was quite difficult to get the back panel closed.

## Pictures
TODO: Add more

![](img/IMG_0008.JPG)
![](img/IMG_0009.JPG)
![](img/archey.png)

## Initial Boot
Once I got everything together, I grabbed a monitor and my Poker II keyboard excited to boot for the first time. Plug everything in, plug the PSU into the wall, hit the power button... Nothing. Lights on the motherboard are on, but nothing happens. No fans spinning, not even the CPU fan. After a few hours of troubleshooting with the help of a good friend of mine and some helpful strangers on IRC, I had determined it was a power problem (next time I should RTFM a little better) and ripped the motherboard out and connected only power to the board and CPU and lo-and-behold it turned on. As I was putting it back together, I realized I had one more standoff installed in the case that was causing the motherboard to short. Goddamnit.

# Home Networking
Before I could go any further... I had to finish setting up the network in my house.

When I did the restoration of my house, I ran all Cat6 wires, but never terminated any of them and set anything up. So I also ordered a 12-port patch panel, and ran to Home Depot to pickup crimpers and connectors and what-not. All of the ethernet in the house was run so that it could all be terminated in the closet of my new addition on the house. The moden is in the living room, since that's where Verizon installed it, so a single cable runs from there to the closet in the back of the house. The upstairs bedrooms run down and back into the closet. The addition also has 2 cables running to the closet.

After wiring the 6 cables into the patch panel, it was time to determine which port was the WAN. The only thing I had in the house that had an ethernet port (aside from the now 100 pound file server being built) was a Raspberry Pi with Arch already installed on it and setup to use a static IP. So, I plugged it into each port one at a time and pinged it from my MacBook. Next I had to determine which port on the patch panel went to the left upstairs bedroom, where the fileserver will reside. *port switching intensifies*

# Installing Arch
Finally, time to install Arch on the server.

I will be following through the Beginners' Guide from the spectacular Arch Wiki. I won't be copy and pasting it all into this writeup, only commenting on sections where I had different input than the default. This includes the contents of my configuration files, at the time of this writeup. I also followed a combination of theese guides for the encrypted zfs root:

https://gist.github.com/codedreality/6006664
https://wiki.archlinux.org/index.php/Installing_Arch_Linux_on_ZFS

## Prerequisites
I'm using a Supermicro X10SL7-F motherboard, which supports UEFI so we're using UEFI.
I created a USB stick installer for Arch.
The first thing I had to do was update the boot order to boot the USB stick in UEFI boot mode.

## Prepare the storage devices
Arch was being installed on /dev/sda, a Mushkin 60GB SSD.
I'll be using dm-crypt to create an encrypted zfs root.

I partitioned using `parted`.

Use GPT, /dev/sda1 is a 512M fat32 /boot. /dev/sda2 is formatted as Solaris Root and is the rest of the disk

```
# mkfs.vfat /dev/sda1
# cryptsetup luksFormat -l 512 -c aes-xts-plain64 -h sha512 /dev/sda2
# cryptsetup luksOpen /dev/sda2 cryptroot
```

## Select a mirror
Generated one with the Mirrorlist Generator, specificly selecting US mirrors only.

## Add demz-repo-archiso and install zfs
```
[demz-repo-archiso]
Server = http://demizerone.com/$repo/$arch
```

```
# mkdir /root/.gnupg
# pacman-key -r 0EE7A126
# pacman-key --lsign-key 0EE7A126
# pacman -Syy archzfs-git
# modprobe zfs
# mkdir -p /etc/zfs
# touch /etc/zfs/zpool.cache
```

## Partition the drive with zfs
```
# zpool create -f -o ashift=12 -o cachefile=/etc/zfs/zpool.cache zroot /dev/mapper/cryptroot
# zfs create zroot/home -o mountpoint=/home
# zfs create zroot/root -o mountpoint=/root
# zfs create zroot/var -o mountpoint=legacy
# zpool set bootfs=zroot zroot
# zpool export zroot
```

## Mount the partitions
```
# zpool import -R /mnt zroot
# mkdir -p /mnt/boot
# mount /dev/sda1 /mnt/boot
# mkdir -p /mnt/var
# mount -t zfs zroot/var /mnt/var
# mkdir -p /mnt/etc/zfs
# cp /etc/zfs/zpool.cache /mnt/etc/zfs/zpool.cache
```

## Generate an fstab
Need to comment out the zroot fs's from `/etc/fstab` so we can keep references of them but, zfs will auto mount them for us.

Also have to add the following for `zroot/var`:
```
# <file system>        <dir>         <type>    <options>             <dump> <pass>
zroot/var              /var          zfs       defaults,noatime      0      0
```

## Add demz-repo-core and install zfs
```
[demz-repo-core]
Server = http://demizerone.com/$repo/$arch
```

```
# mkdir /root/.gnupg
# pacman-key -r 0EE7A126
# pacman-key --lsign-key 0EE7A126
# pacman -Syy zfs-git
# modprobe zfs
# systemctl enable zfs.target
# systemctl start zfs.target
```

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

## Create an initial ramdisk environment
Edit `/etc/mkinitcpio.conf` and make hooks like like so:
```
HOOKS="base udev autodetect modconf block keyboard encrypt zfs filesystems"
```

## Install and configure a bootloader
We're using UEFI, so here are the commands I had to use.

```
# pacman -S dosfstools efibootmgr gummiboot
# gummiboot --path=/boot install
# cat | sudo tee /boot/loader/entries/arch.conf >/dev/null
title   Arch Linux
linux   /vmlinuz-linux
initrd  /initramfs-linux.img
options cryptdevice=/dev/sda2:cryptroot zfs=zroot spl.spl_hostid=<hostId> quiet rw
<C-d>
# cat | sudo tee /boot/loader/loader.conf >/dev/null
default arch
timeout 5
<C-d>
```

## Unmount the partitions and reboot
```
# exit
# umount /mnt/boot
# zfs umount -a
# zpool export zroot
```

## After the first boot
```
# zpool set cachefile=/etc/zfs/zpool.cache zroot
# zfs set atime=off zroot
# zfs set compression=lz4 zroot
```

## Install cronie and zpool scrub cronjob
```
# pacman -Syy cronie
# crontab -e
30 19 * * 5 zpool scrub zroot
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

I then secured SSH by following [this guide](https://stribika.github.io/2015/01/04/secure-secure-shell.html).

### Add my user
```
# useradd -m -G wheel -s /bin/bash marc
# passwd marc
```

### Configure iptables
I followed [this guide](https://wiki.archlinux.org/index.php/Simple_stateful_firewall) to setup a simple stateful firewall. For now, only port 22 is allowed.

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

### Install mosh
```
$ yaourt mosh
$ sudo iptables -A UDP -p udp --match multiport --dports 60000:61000 -j ACCEPT
$ sudo iptables-save | sudo tee /etc/iptables/iptables.rules
```

### Configure Time
Realized my time was off.. Used [this](https://wiki.archlinux.org/index.php/Time) guide to configure `ntpd`. Had to sync my hwclock.

### Install powerpanel
I bought a CyberPower UPS for this server build, not going to risk losing power and losing data. Or getting hit by lightning, again.
Followed [this guide](https://wiki.archlinux.org/index.php/CyberPower_UPS) and just used the default settings.

### Setup encrypted zpool with storage drives
```
Now we still have the 8x 3TB drives that haven't been touched. Time to put them into a raidz2 array so we can start filling them.

I'll be encrypting the drives, and using a passphrase and a keyfile for decrypting at boot with `crypttab`.

In the future I plan on chaning how I'm doing my keys, they'll be on a USB key that only gets plugged in for booting. Would also like to use GPG to decrypt the keyfiles as well, as they're just plaintext files.

# parted -a optimal /dev/sd{b,c,d,e,f,g,h,i} mklabel gpt mkpart primary 2048s 100% p
# cryptsetup --cipher aes-xts-plain64 --key-size 512 --hash sha512 --iter-time 5000 --use-urandom --verify-passphrase luksFormat /dev/sd{b,c,d,e,f,g,h,i}1
# dd bs=1 count=4096 if=/dev/urandom of=/etc/keyfile iflag=fullblock
# cryptsetup luksAddKey /dev/sd{b,c,d,e,f,g,h,i}1 /etc/keyfile
# vim /etc/crypttab
data0 /dev/sdb1 /etc/keyfile
data1 /dev/sdc1 /etc/keyfile
data2 /dev/sdd1 /etc/keyfile
data3 /dev/sde1 /etc/keyfile
data4 /dev/sdf1 /etc/keyfile
data5 /dev/sdg1 /etc/keyfile
data6 /dev/sdh1 /etc/keyfile
data7 /dev/sdi1 /etc/keyfile
# zpool create -m /data -o ashift=12 tank raidz2 dm-name-data0 dm-name-data1 dm-name-data2 dm-name-data3 dm-name-data4 dm-name-data5 dm-name-data6 dm-name-data7
```

### Setup Time Machine backups
```
$ yaourt netatalk
$ sudo zfs create data/timemachines
$ sudo zfs create data/timemachines/marc
$ sudo zfs set quote=500G data/timemachines/marc
$ sudo chown -R marc:marc /data/timemachines/marc
$ sudo vim /etc/afp.conf
[Global]
mimic model = RackMac6,106
log level = default:warn
log file = /var/log/afpd.log
hosts allow = 192.168.1.0/24

[Time Machine]
path = /data/timemachines/marc
valid users = marc
time machine = yes
$ sudo iptables -A UDP -p udp --dport mdns -d 224.0.0.251 -j ACCEPT
$ sudo iptables -A TCP -p tcp --dport afpovertcp -j ACCEPT
$ iptables-save | sudo tee /etc/iptables/iptables.rules
$ sudo systemctl enable netatalk.service
$ sudo systemctl start netatalk.service
```
