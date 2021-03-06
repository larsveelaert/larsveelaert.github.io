---
title: 'HackOTG (v1.0): A universal, portable, security-platform'
layout: post
categories:
  - HackOTG
tags:
  - embedded
  - ethernet
  - OTG
  - rpi
  - usb
---
![HackOTG]({{ "/assets/HackOTG.jpg" | relative_url }})

_When demonstrating a possible attack vector to a friend or a client, you want something standalone that is not influenced by your day-to-day tasks. Many have a persistent [Kali Linux](https://www.kali.org/) on a bootable USB-drive exactly for this purpose. There are also many attack-vectors which require physical devices. Exploits like [DNS-tunneling](https://larsveelaert.github.io/2017/10/03/hack-get-free-wifi-on-paid-access-hotspots/), HID-attacks and [BadUSB](https://www.topsec.com/it-security-news-and-info/what-is-badusb-and-should-i-be-scared) are only a couple of examples._

You can buy specific devices like the [USB rubber ducky](https://hakshop.com/products/usb-rubber-ducky-deluxe) to perform these attacks in an easy, portable, quick way. Apart from being quite costly, you can&#8217;t carry them all in your everyday backpack and they often require to be reprogrammed for a different attack on the same vector&#8230; So having multiple will only make it worse.

I&#8217;ve ported a lot of my physical exploits and some wifi-exploits to my Android device. It&#8217;s disguised, an all-in-one and you&#8217;ll always have it on you. An OTG-cable in my backpack and I&#8217;m ready to go&#8230; But often a lot of these exploits require a custom kernel or some modules to be able to function. Many tweaks are also device specific. When there is an update, a lot of unexpected things can happen. Not the best solution.

So I was searching something open-source, portable, inexpensive, expendable, easy-to-mod and easy to interface with and change it behavior and I found this:

![Rpi Zero on adafruit]({{ "/assets/Rpizero.png" | relative_url }})

It is essentially a small stand-alone computer, running Linux of an SD-card. What is cool about this model is, that it has WiFi and Bluetooth and a board-setup configured to support USB-gadget or OTG mode. Which can change one of the micro-USB ports in whatever USB-device that you want (HID, Ethernet Dongle, Mass-storage, MIDI, &#8230; ). It is powered with a normal 5v micro-USB cable plugged in a USB-device or wall-adapter.

We can set it up as a headless device that we can ssh into over a WiFi-signal or even easier, making it emulate an ethernet device. The rest of this article will describe how to get it to emulate an Ethernet to USB controller and SSH&#8217;ing into it without ever attaching a display.

## Preparing the SD-card

The software that is best adapted to the hardware is [Raspbian](https://www.raspberrypi.org/documentation/raspbian/). You can try to do the same steps on other ARM OS-distro&#8217;s, but from my experience they lack a lot of support for OTG. The idea is to make a very-easy-to-recreate setup so if it is not your favourite flavour of linux, no worries.

There are [many ways to install Raspbian](https://www.raspberrypi.org/documentation/installation/installing-images/), but you basically download the [diskimage](https://en.wikipedia.org/wiki/Disk_image) and write it to the SD-card. if your SD-card is bigger than the original disk-image, expand the partition to its full size. In this article you&#8217;ll find a guide how to do this on Linux.
  
A SD-card in a raspberry pi consists of 1 small FAT-partition with boot files and 1 large EXT4-partition. Recreating the partition table and copying the right files on each will also give you a working system (but Raspbian gives you a ready disk-image).

**Download the Raspbian Lite-image:** (You don&#8217;t need te full one)

![Raspbian Download]({{ "/assets/raspian_download.png" | relative_url }})

<pre>wget https://downloads.raspberrypi.org/raspbian_lite_latest
unzip raspbian_lite_latest #will unzip an .img file</pre>

**Flash your SD-card:**

Plug your SD-card in your pc with an card-reader. Your SD-card size and type matter, but are not that important. You want a fast one (indicated by its class, 10 is best) and anything above 2 gb will work well if using the lite-image. If you deploy this somewhere and it has to record or log something, make sure that it has disk-space to do so. At the time of writing a  quality SD-card of 32 gb is 20 EUR, so that would bring the total cost to ~30 EUR excluding cables (which you probably have too much of).
  
<span style="text-decoration: underline;">Double-check before running these commands!</span>

<pre>fdisk -l #lookup the device name of your SD-card (ex. /dev/mmcblk1)</pre>

<pre class="output">Disk <strong>/dev/mmcblk1</strong>: 29.8 GiB, 32026656768 bytes, 62552064 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x00000000

Device    Boot Start End      Sectors  Size  Id Type
/dev/mmcblk1p1 8192  62552063 62543872 29.8G c  W95 FAT32 (LBA)</pre>

<pre>dd bs=4M if=2017-09-07-raspbian-stretch-lite.img of=/dev/mmcblk1 conv=fsync #change /dev/mmcblk1</pre>

<pre class="output">442+1 records in
442+1 records out
1854590976 bytes (1.9 GB, 1.7 GiB) copied, 99.8628 s, 18.6 MB/s</pre>

**Resize partition to full disk: **_(Optional)_

<pre>fdisk /dev/mmcblk1 #change /dev/mmcblk1 to your device</pre>

<pre class="output">Welcome to fdisk (util-linux 2.30.2).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.
<strong>Command (m for help): p</strong>
Disk /dev/mmcblk1: 29.8 GiB, 32026656768 bytes, 62552064 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x11eccc69
Device Boot Start End Sectors Size Id Type
/dev/mmcblk1p1 8192 93813 85622 41.8M c W95 FAT32 (LBA)
/dev/mmcblk1p2 94208 62552063 62457856 29.8G 83 Linux
<strong>Command (m for help): d</strong>
<strong>Partition number (1,2, default 2): 2</strong>
Partition 2 has been deleted.
<strong>Command (m for help): n</strong>
Partition type
 p primary (1 primary, 0 extended, 3 free)
 e extended (container for logical partitions)
<strong>Select (default p): p</strong>
<strong>Partition number (2-4, default 2): 2</strong>
<strong>First sector (2048-62552063, default 2048): 94208</strong>
Last sector, +sectors or +size{K,M,G,T,P} (94208-62552063, default 62552063):
Created a new partition 2 of type 'Linux' and of size 29.8 GiB.
Partition #2 contains a ext4 signature.
<strong>Do you want to remove the signature? [Y]es/[N]o: n</strong><strong>
Command (m for help): w</strong>
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.</pre>

At this point you will have a working operating system.

## Configuring the device

Now that you having a working image, it is time to change some files with settings to make you able to connect to it. This will be achieved by configuring the OTG-port (micro-usb port marked with &#8220;USB&#8221;) to emulate an Ethernet-to-USB dongle. With a working connection, we will be able to log in through SSH.

**Mount both partitions:**

<pre>mkdir boot
mount /dev/mmcblk1p1 boot
mkdir root
mount /dev/mmcblk1p2 root</pre>

**Setup USB-gadget, Ethernet-to-USB:**

Add a line to boot/config.txt, using this command;

<pre>echo 'dtoverlay=dwc2' &gt;&gt; boot/config.txt</pre>

After the command &#8216;rootwait&#8217; in boot/cmdline.txt add the following command (keep it as one line);

<pre># boot/cmdline.txt
... rootwait modules-load=dwc2,g_ether ...</pre>

This will load the kernel-module needed and emulate a Ethernet-to-USB device on the USB port. You can also change this after the device is booted, but we have no other way of accessing the device, so we want this behavior on boot.

**Configure a static IP:**

Open the file root/etc/network/interfaces and add the following snippet to the end of the file:

<pre><span class="pln"># root/etc/network/interfaces
allow</span><span class="pun">-</span><span class="pln">hotplug usb0
</span><span class="pln">iface usb0 inet </span><span class="kwd">static
</span><span class="pln">address </span><span class="lit">192.168</span><span class="pun">.</span><span class="lit">7.2
</span><span class="pln">netmask </span><span class="lit">255.255</span><span class="pun">.</span><span class="lit">255.0
</span><span class="pln">network </span><span class="lit">192.168</span><span class="pun">.</span><span class="lit">7.0
</span><span class="pln">broadcast </span><span class="lit">192.168</span><span class="pun">.</span><span class="lit">7.255
</span><span class="pln">gateway </span><span class="lit">192.168</span><span class="pun">.</span><span class="lit">7.1
</span></pre>

If the device is connected with its emulated ethernet-device, we will know its IP without using a DHCP-server or discovery-service.

**Enable SSH-server (headless way):**

SSH-server was enabled on the early versions but there was [a security issue](https://www.raspberrypi.org/documentation/remote-access/ssh/) for many people who were not aware of this. To make it easy to enable this feature you can put an empty file named &#8216;ssh&#8217; on the boot partition and it will enable it for you (the file will be gone after).

<pre>touch boot/ssh</pre>

**Unmount the SD-card:**

<pre>sync
unmount boot root</pre>

## Using the device

If everything is set up correctly, you can use a micro-USB to USB-A (regular charging cable) to connect your device with the &#8220;USB&#8221; marked port to your computer (or any other device with a terminal). The connection will give both a 5V power and a data-connection to your Pi. After plugging the device in, the ACT light will start blinking if the SD-card has a bootable operating system on it. The first time the boot up process will take longer, but normally this takes about 10 seconds.

![HackOTG]({{ "/assets/HackOTG.jpg" | relative_url }})

On Linux, you can check your interfaces by issuing the command:

<pre>ifconfig</pre>

If everything is working properly, you&#8217;ll see this new interface;

<pre class="output">...
usb0: flags=4163&lt;up,broadcast,running,multicast&gt; mtu 1500
     inet6 fe80::a8ac:4ff:fe31:3a2e prefixlen 64 scopeid 0x20
     ether aa:ac:04:31:3a:2e txqueuelen 1000 (Ethernet)
     RX packets 28 bytes 3312 (3.2 KiB)
     RX errors 0 dropped 0 overruns 0 frame 0
     TX packets 5 bytes 734 (734.0 B)
     TX errors 0 dropped 0 overruns 0 carrier 0 collisions 0
...&lt;/up,broadcast,running,multicast&gt;</pre>

To be able to SSH to the device we will need an IP ourselves. There is no DHCP-server on this 2-man network so we will give us one ourselves (in the same netmask). Use this command: (You&#8217;ll have to do this every time you plug in the device)

<pre>ifconfig usb0 192.168.7.3</pre>

Now you can SSH to this device with the default credentials (user: pi, pass: raspberry):

<pre>ssh pi@192.168.7.2</pre>

## What next?

Now that we have the basics figured out, in the next version we will hack the hardware to make the device include a USB-port so it is truly portable.
  
In the rest of the series we will start building our universal hacking/debugging tool so we can connect to it over WiFi and use its emulation capabilities to demonstrate physical attack-vectors.

_You can find the next one here:_

<blockquote class="wp-embedded-content" data-secret="HpdD86nrmQ">
  <p>
    <a href="/2017/10/10/hackotg-v1-1-integrating-the-cable-for-more-portability/">HackOTG (v1.1): Integrating the cable for more portability</a>
  </p>
</blockquote>
