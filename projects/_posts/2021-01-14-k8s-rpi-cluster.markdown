---
layout: post
title:  "Raspberry pi k8s cluster - Part 1"
date:   2021-01-14 23:14:00 +0100
categories: jekyll raspberry rpi k8s synology kubernetes
tag: k8s synology kubernetes iscsi pxe
---
## Raspberry pi cluster
So I decided to build my own raspberry pi kubernetes cluster in an effort to make it possible to shutdown my home server when I'm not working on it. The cluster should consume less power then my current always on intel system with 2 gpus. 
My home monitoring system shows an idle power usage of around 245W for my server, which is rather high. I didn't do my best to make it green, so this new system should be greener, leaner and meaner.
It should be sort-of redundant to failure and capable of hosting my docker containers which are now on my always on system.
I started out with three raspberry pi 4 nodes with 4 GB of ram and some additional stuff to assemble  a cluster. I got most of the parts scattered around different shops, so here is a list.

* [Rpi 4 nodes][nodes]{:target="_blank"}
* [Simple 4 port USB powered switch][usb-switch]{:target="_blank"}
* [USB Power supply - 63W][anker-psu]{:target="_blank"}
* [Cat6a Ethernet cable - 0.5m][ethernet-cable]{:target="_blank"}
* [USB C cable][usb-cable]{:target="_blank"}
* [Rpi Rack enclosure][rpi-rack]{:target="_blank"}
* [32 GB Sd-cards][sd-cards]{:target="_blank"}

## Choosing an OS and booting it
I already have several Raspberry Pi's in house for quite some time, so I am familiar with raspbian. Since I'm not a big fan of SD cards and especially not of broken SD cards, I've always only used the sd card to create a boot partition and then start the kernel with a remote nfs root filesystem.
This method has served me well for quite some years. The only drawback is you need the SD card to boot the system. As of model 4 it is now possible to PXE boot or even USB boot the Pi's without the use of an additional SD card. I decided to give it a go and try to boot the systems using PXE instead of the traditional methods.

I already have tftp and PXE configured in my home environment, so I just needed to figure out the correct path that get's used when the pi asks for it's configuration. I used [this][rpi-pxe-config] documentation to enable pxe on all my pi's. Mine all had an old eeprom, so I first had to update the eeproms, reboot them and then enable the usb boot. I used the same SD card for all 3 pi's and that seems to work fine.

Afterwards I discovered it is also possible to set the network boot option using raspi-config. More details to be found [here](https://www.raspberrypi.org/documentation/hardware/raspberrypi/bcm2711_bootloader_config.md)


The guide for upgrading the firmware can be found [here](https://www.raspberrypi.org/documentation/hardware/raspberrypi/booteeprom.md), below is the short version I used on each pi afther the first was ready.

### Manually checking if an update is available
Running the rpi-eeprom-update command with no parameters indicates whether an update is required. An update is required if the version of the most recent file in the firmware directory (normally /lib/firmware/raspberrypi/bootloader/critical) is newer than that reported by the current bootloader. The images under /lib/firmware/raspberrypi/bootloader are part of the rpi-eeprom package and are only updated via apt upgrade.

```
sudo rpi-eeprom-update
```

If an update is available, you can install it using:

```
sudo rpi-eeprom-update -a
sudo reboot
```

## Installing the system
There are several tutorials available on the net, explaining how to setup pxe server and boot your pi from the server.
I used a combination of several tutorials to setup my system.

* [Linuxhit tutorial](https://linuxhit.com/raspberry-pi-pxe-boot-netbooting-a-pi-4-without-an-sd-card/)
* [Hackaday tutorial](https://hackaday.com/2019/11/11/network-booting-the-pi-4/)
* [Ferdinand Keil](https://www.ferdinand-keil.com/network-booting-rpi4-from-centos7.html)
* [Code Strain](https://codestrian.com/index.php/2020/02/14/setting-up-a-pi-cluster-with-netboot/)
* [VirtuallyGhetto](https://www.virtuallyghetto.com/2020/07/two-methods-to-network-boot-raspberry-pi-4.html)

First I had to figure out the serial number of the pi's. You can do this with this command on each pi.
```
grep  Serial /proc/cpuinfo  | tail -c 9
05a07819
```

After gathering the serial number, I setup a file structure on my tftpboot server, to be able to boot each of the pi's directly using tftpboot.
```
rpi4
rpi4/raspbian
rpi4/raspbian/05a07819
rpi4/raspbian/18d48af8
rpi4/raspbian/5d54ebad
rpi4/ubuntu
rpi4/ubuntu/05a07819
...
05a07819 -> rpi4/ubuntu/05a07819
18d48af8 -> rpi4/ubuntu/18d48af8
5d54ebad -> rpi4/ubuntu/5d54ebad
```
Notice the rpi4/ubuntu and rpi4/raspbian folders, this makes it possible to quickly switch between raspbian and ubuntu by just switching the symlink to the other location and rebooting the Pi.
You do need a seperate root filesystem offcourse, but it's obvious you won't be mixing those together.

There is a difference in boot folders between ubuntu and raspbian. Be sure to copy all the files from the raspbian boot folder image to the raspbian boot folder and the same for ubuntu if you're going to use this.
I initially struggled with getting my ubuntu up and running. It was stuck in the color radiant screen and didn't continue the boot.
In the end I re-copied all files from the ubuntu server boot image to the boot folder and then the kernel did load.
For ubuntu I changed the following files:
config.txt :
```
[pi4]
max_framebuffers=2
[pi2]
kernel=uboot_rpi_2.bin
[pi3]
kernel=uboot_rpi_3.bin
[all]
arm_64bit=1
device_tree_address=0x03000000
kernel=vmlinuz-5.8.0-1010-raspi
initramfs initrd.img-5.8.0-1010-raspi followkernel
dtparam=sd_poll_once
dtparam=eee=off
dtparam=pwr_led_trigger=cpu0
dtparam=act_led_trigger=cpu2
enable_uart=1
cmdline=cmdline.txt
include syscfg.txt
include usercfg.txt
```
syscfg.txt :
```
enable_uart=1
dtparam=audio=on
dtparam=i2c_arm=on
dtparam=spi=on
cmdline=cmdline.txt
```

usercfg.txt :
```
cmdline=iscsi.txt
```

iscsi.txt :
```
net.ifnames=0 dwc_otg.lpm_enable=0 console=serial0,115200 console=tty1 ip=dhcp cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory root=LABEL=RPI4-NODE1 rootfstype=ext4 elevator=deadline rootwait fixrtc ISCSI_INITIATOR=iqn.1993-08.org.debian:01:dca632b27398 ISCSI_TARGET_NAME=iqn.2000-01.com.synology:synology.Rpi4-node1.7335afa52f ISCSI_TARGET_IP=192.168.178.147 ISCSI_TARGET_PORT=3260 cloud-init=disabled
```

These are the options for iscsi booting the pi's from my synology.

I created an iscsi volume on the synology, mounted that on one of my servers and dumped the entire contents of the ubuntu server SD image to the iscsi disk. After the dump, I mounted the image and altered fstab to reflect the iscsi disk as root device. I also created a pxe folder in /boot, so I can change boot settings directly from the pi.
All that was left now was setting the hostname of the pi and changing /etc/hosts

```
vim /etc/fstab
...
192.168.178.147:/volume1/tftpboot/rpi4/ubuntu/05a07819 /boot/pxe nfs defaults,ro,nolock 0 0
UUID="add52206-1093-4aa9-8b2d-d2b7e2e26a56"	/	ext4	relatime,lazytime,_netdev,x-systemd.requires=iscsid.service 0 1

vim /etc/hosts
...
127.0.1.1 rpi4-node1
...

vim /etc/hostname
rpi4-node1
```

## Reference list of files /boot/pxe (or tftpboot folder)

```
ls 
bcm2708-rpi-b.dtb       bcm2708-rpi-zero-w.dtb  bcm2710-rpi-3-b-plus.dtb  config.txt                   iscsi.txt     System.map-5.4.0-1015-raspi
bcm2708-rpi-b-plus.dtb  bcm2709-rpi-2-b.dtb     bcm2710-rpi-cm3.dtb       fixup4.dat                   kernel7l.img  usercfg.txt
bcm2708-rpi-cm.dtb      bcm2710-rpi-2-b.dtb     bcm2711-rpi-4-b.dtb       initrd.img-5.8.0-1008-raspi  start4.elf    vmlinuz-5.8.0-1008-raspi
bcm2708-rpi-zero.dtb    bcm2710-rpi-3-b.dtb     cmdline.txt               initrd.img-5.8.0-1010-raspi  syscfg.txt    vmlinuz-5.8.0-1010-raspi
```
As you can see, most of the files were copied from the ubuntu image, but certainly not all of them.
These were the only ones required to boot the system. I debugged my tftp server to make sure which files were needed and added the subsequent files as they were required.
In the end I also added all of the dtb files, if at some point I might use the same setup for my older pi's


## Making the system work

I had some issues with the default ubuntu configuration so I had to remove some trouble causing packages.
The /lib/modules/kernel-version folder gets bind mounted as a virtual filesystem by the cloud-init in the initrd phase.
It's not possible to unmount the FS, as the modules are loaded from the filesystem. You can't upgrade the kernel, since the new kernel modules will be default copied in the tmpfs and will be wiped at reboot. Long story short, I fixed it like this:
```
apt remove cloud-init cloud-guest-utils cloud-initramfs-copymods cloud-initramfs-dyn-netconf
```

When I need to upgrade the kernel for one of the systems, I just run the upgrade and everything gets installed in the iscsi drive.
Afterwards I copy over the initrd and vmlinuz to the /boot/pxe folder and alter the config.txt file

## Making copies for the other pi's

Now I had a working system, it's time to make copies for the other pi's.
I created extra iscsi volumes for the 2nd and 3rd pi and then ran the following to create sparse clones of the drive.
```
ssh synology -l root
cd /volume1/\@iSCSI/LUN/BLUN_THICK/
dd if=./bfe86945-d487-46ff-bfa2-3917e6af3ea0/RPI4-NODE1_00000 of=./154eaf5f-1674-419d-8a1c-7fc2487bb9ca/RPI4-NODE3_00000 bs=512 conv=sparse &
while true ; do clear; ls -lash 154eaf5f-1674-419d-8a1c-7fc2487bb9ca/ ; pkill -USR1 dd ; sleep 5 ; done
```
After the drives were copied, I initially iscsi mounted them on my first pi, so I could alter the fstab and iscsi files.

You need to make sure the /etc/iscsi/initiatorname.iscsi contains an unique name. I used the mac address of each pi for creating the unique address.

After these configuration changes were made, I applied policies on the synology allowing each pi to only mount their volume.
I noticed before applying this policy the volumes were detected on all pi's during a reboot and that stopped the boot from the subsequent pi's as the synology default disables concurrent writes to an iscsi volume. The same rule applies when booting the system. The initiator name from your boot line has to match the initiator in the configuration file after boot.
```
cat /etc/iscsi/initiatorname.iscsi
InitiatorName=iqn.1993-08.org.debian:01:dca632b27398
```
With these policies in place it is possible to reboot all 3 pi's at the same time and don't get kernel timeouts or iscsi timeouts.
I have a nvme cache in front of my raid volume in the synology. The real bottleneck at boottime is my 1gbit interface.
Overall I'm very pleased with the performance of the iscsi volumes and since my nas is always on, I don't mind mounting the pi's over iscsi and not nfs.
Nfs gives you a whole lot of troubles when you're using ubuntu. You can't use snap for starters, you can't mount your home folders correctly and don't even think about using docker or podman to play with some containers on an nfs volume. I switched from nfs to iscsi simply for this reason and don't regret making the decission (yet)


This concludes my Part 1 of setting up my k8s pi cluster. 

In the next post I'll discuss my experience and tests with different kubernetes flavors.

As a reference below is my initial nfs configuration. Feel free to try it, but be warned, you will run into troubles!.

## Alternate NFS server configuration

If you want to boot from an nfs server, this is an example configuration that should get you started.

cmdline.txt :
```
console=serial0,115200 console=tty root=/dev/nfs nfsroot=192.168.178.147:/volume1/microk8s/rpi4-05a07819-ubu,vers=3 rw ip=dhcp rootwait elevator=deadline cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory net.ifnames=0 dwc_otg.lpm_enable=0 fixrtc cloud-init=disabled
```


[nodes]: https://www.raspberrystore.nl/PrestaShop/raspberry-pi-v4/228-raspberry-pi-4b4gb-765756931182.html
[usb-switch]: https://www.bol.com/nl/p/renkforce-netwerk-switch-4-poorten-1-gbit-s/9200000095488552/?s2a=
[anker-psu]: https://www.bol.com/nl/p/anker-powerport-speed-5-poorts-2x-quick-charge-63w-thuislader-zwart/9200000077301036/?s2a=
[ethernet-cable]: https://www.aliexpress.com/item/SAMZHE-Cat6A-Ethernet-Cable-Ultrafine-Cat-6-UTP-Ethernet-Patch-Cable-Slim-RJ45-Computer-XBox-Networking/33026678139.html
[usb-cable]: https://www.aliexpress.com/item/ZNP-USB-Type-C-Cable-For-Samsung-S10-Huawei-P30-Pro-Fast-Charge-Type-C-Mobile/32997902868.html
[rpi-rack]:https://www.raspberrystore.nl/PrestaShop/behuizingen/257-stapelbare-behuizing-helder.html
[sd-cards]: https://www.aliexpress.com/item/Original-SanDisk-Micro-SD-Card-128GB-32GB-64GB-16GB-Ultra-TF-card-Class-10-Memory-Card/33036949589.html
[rpi-pxe-config]:https://www.raspberrypi.org/documentation/hardware/raspberrypi/bootmodes/net_tutorial.md

