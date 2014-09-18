---
layout: post
title: "CentOS 7 and older HP RAID controllers"
description: ""
category: Linux
tags: []
---
{% include JB/setup %}
### Installing CentOS 7 on older HP ProLiants

No drives detected in the installer? No drives detected after install?

If you're looking to get straight to the point, skip to Installing CentOS7 + Enabling HPSA, otherwise there's a bit of background below.

### Background

So ... wanting to run <a href="https://github.com/stec-inc/EnhanceIO">EnhanceIO</a> requires the linux 3.7+ kernel and running CentOS 6.5 (2.6) on our production mongodb cluster left me with a bit of work ahead. The most obvious path to take from my perspective was CentOS 7, which is based on the sources of the recently release RHEL7. The upgrade path: Take one of our secondaries out of the replica set, upgrade it to CentOS 7 which has the 3.10 kernel and go from there. 

*More on EnhanceIO and what I'm doing with it in a later post...*

As if my week hadn't already been muddled by having to plan a logistical nightmare!

Anaconda boots, we choose our language, and boom. It can't find any drives?!

On this particular HP Proliant it has a P400i RAID controller, now I've had issues with CentOS 6 and the B110i before but that was due to it being a terrible excuse for a RAID controller - this was not showing the same symtoms.

A quick google brought up <a href="http://serverfault.com/questions/611182/centos-7-x64-and-hp-proliant-dl360-g5-scsi-controller-compatibility" target="_blank" >this StackOverflow article</a>.

**PROBLEM:** RHEL7 removes the the the CCSIS driver and you need to load the kernel component correctly before it can see any drives.

There are two parts to this install:

1. Loading the kernel component for the installer (or rather allow HPSA to load any old driver)

2. Altering the bootloader to ensure that on boot the kernel uses the same HPSA directives, otherwise after install it'll boot and guess what... it won't see your drives!

### Enabling HPSA Allow Any on the Installer

I used the CentOS 7 minimal ISO but the DVD should work fine too.

1. When the installer boots to the options screen **hit the tab key** to get the command line up. 

2. Enter the following on the end and hit return:

	```
	hpsa.hpsa_simple_mode=1 hpsa.hpsa_allow_any=1
	```

When the installer boots up, you should see your logical volume and use it as an install destination. Select it and install your system.

When it reboots though, if you let it get to that stage, you'll end up in emergency mode because it won't find your drive again.

### HPSA Allow Any on Boot

It makes sense that on the freshly installed  system, the kernel modules that we needed to load for the CCISS drivers won't load on boot.

We can fix this by regenerating the GRUB boot file, although to do this we need to mount the filesystem and intern we need to reboot into rescue mode (which is why we still need the CD/DVD).


1. Reboot the machine with the disk in and as before on the options screen **hit the tab key** to get the command line up.

2. This time we'll want to boot into rescue mode **with** the hpsa module set to allow our CCISS driver, type and hit enter:

	```
	rescue hpsa.hpsa_simple_mode=1 hpsa.hpsa_allow_any=1
	```

3. It'll start to boot and hopefully you'll be prompted about mounting a filesystem, just hit continue at this point and then okay when it mentions something about chroot'ing to /mnt/sysimage. (note: It may when loading want to check your disks which it mentions you can skip by pressing ESC... if you want.)

4. Hopefully you'll have reached the rescue mode shell, first thing we need to do is change the root to the system image like it mentioned:

	```
	chroot /mnt/sysimage
	```

5. Next we need to append the module directives to the GRUB_CMDLINE_LINUX variable like so:
	
	```
	GRUB_CMDLINE_LINUX="rd.lvm.lv=centos/swap crashkernel=auto  rd.lvm.lv=centos/root vconsole.font=latarcyrheb-sun16 vconsole.keymap=uk rhgb quiet hpsa.hpsa_simple_mode=1 hpsa.hpsa_allow_any=1"
	```

6. Make the GRUB file (this is the bit that actually changes the bootloader):

	```
	grub2-mkconfig -o /boot/grub2/grub.cfg
	```

7. Reboot and you should be good!

### Thoughts

Although not ideal, this seems to be the only way I've managed to piece together so far. Really it didn't take that long to find out the correct information but hopefully it'll save someone some time. I hate it when there's relatively no information regarding isolated issues like this!

I guess the underlying issue here is the drivers are deprecated and are the hardware is ex-enterprise being over a few years old now.

If you do have any issues, feel free to tweet me and I'll be happy to help. @jordanisonfire



