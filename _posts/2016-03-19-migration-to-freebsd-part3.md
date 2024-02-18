---
title: Migrating to FreeBSD - Part 3
tags: [epijunkie.com, series, freebsd, solaris, zfs, stmfadm, fiber channel, lun]
style: border
color: primary
description: Part 3 in a series to migrate to FreeBSD.
---

Migrating to FreeBSD from Solaris 11: Part 3 - Importing ZVols for serving via Fiber Channel
==========================

During [this transition to FreeBSD for my storage host](http://justinholcomb.me/blog/2016/02/28/migration-to-freebsd-part1.html), the ZVols being serve through Fiber Channel need to be migrated and then reconfigured on FreeBSD.

This part of the series will cover moving the block devices from Solaris to FreeBSD. This block devices are being served over Fiber Channel using ZVols backing. The goal is to retain the GUIDs necessary so that the end devices continue to operate correctly and all of this without the useful `zfs send | zfs recv` commands due to the difference in `zfs` [6 and 5] and `zpool` [37 and 5000] version respectively.

This process starts by gathering some key information from the Solaris machine. This information includes the GUID, the exact byte size of the ZVol, and then the actual data. Then on the FreeBSD host, the FiberChannel target mode needs to be turned on, the ZVols create, the data copied, and the Fiber Channel LUNs configured.

First grab the GUID and size from the `stmfadm` utility (LU Name and Size) from the Solaris machine.

```
root@solarishost:~# stmfadm list-lu -v
LU Name: 600144F010010600000055577FAA0002
Operational Status : Offline
Provider Name : sbd
View Entry Count : 1
Data File : /dev/zvol/rdsk/tank/zvols/esxi-shared
Meta File : not set
Size : 6442450944
Block Size : 512
Management URL : not set
Vendor ID : SUN
Product ID : COMSTAR
Serial Num : not set
Write Protect : Disabled
Write Cache Mode Select: Enabled
Writeback Cache : Enabled
Access State : Active
```

As a cautionary practice, verify the ZVol size:

```
root@solarishost:~# zfs get -p volsize tank/zvols/esxi-shared
NAME PROPERTY VALUE SOURCE
cantrill/zvols/unit8esxi volsize 6442450944 local
```

Next save the contents of the ZVol using `dd`.

`root@solarishost:~# dd if=/dev/zvol/rdsk/tank/zvols/esxi-shared of=/newtank/esxi-shared.zvol.raw.img`

For those uncomfortable with `dd`, the file extension does not matter. I used `.zvol.raw.img` extension because it is arbitrary/relevant to me. This might be further relevant in a few years when I still have not deleted this file and stumble upon it.

After transferring the files to the FreeBSD host, the next step is to configure the it. Lets start off by recreating the ZVol. The default volume blocksize was defaulting to 8k which was causing some issues with my ESXi datastore. If the blocksize is specified to 512 byte, ESXi will work as expected and probably performing a little better as well depending on the underlying harddisks blocksize.

`root@freebsdhost: ~# zfs create -V 6442450944 -o volblocksize=512 newtank/zvols/esxi-shared`

Then write the image file back to the ZVol.

`root@freebsdhost: ~# dd if=/newtank/esxi-shared.zvol.raw.img of=/dev/zvol/newtank/zvols/esxi-shared`

It is also suggested to set a ZFS property to keep track of the LU device-id. This way if the OS drive dies, the necessary information to recreate it the LUN remains with the data. This is something inspired by [Allan
Jude](http://www.allanjude.com).

`root@freebsdhost: ~# zfs set lu:device-id=600144F010010600000055577FAA0002 newtank/zvols/esxi-shared`

The last stage of this is to setup the FreeBSD machine as a Fiber Channel target and configure the ZVol to be presented as a LUN.

This guide assumes the use of a Qlogic card. They are remarkably cheap, for under $100 on eBay I was able to create a direct connection network for four initiators and one target. Putting a Qlogic Fiber Channel card into target mode does involve recompiling the kernel which is straight-forward.

First create a new kernel configuration file, below is the recommended way. This method keeps the latest GENERIC kernel as the base with the current GENERIC as packaged with your release. This keeps the configuration current with future FreeBSD improvements to the GENERIC configuration and also keeps the kernel configuration safe from accidental deletion when updating the source.

```
root@freebsdhost: ~# mkdir ~/kernels

root@freebsdhost: ~# touch ~/kernels/FCTARGETMODE

root@freebsdhost: ~# ln -s /root/kernels/FCTARGETMODE /usr/src/sys/amd64/conf/

Choose your favorite text editor and add the following to the new kernel configuration file.

~/kernels/FCTARGETMODE
<pre class="programlisting">include GENERIC
ident FCTARGETMODE

device          ispfw                   #Firmware for QLogic HBAs
options         ISP_TARGET_MODE         #Required for Target Mode</pre>
Then build, install, and reboot.

root@freebsdhost: ~# cd /usr/src

root@freebsdhost: ~# make buildkernel KERNCONF=FCTARGETMODE

root@freebsdhost: ~# make installkernel KERNCONF=FCTARGETMODE

root@freebsdhost: ~# reboot
```

Verify the new kernel is running:

```
justin@freebsdhost: ~$ uname -i
FCTARGETMODE
```

Next configure the ZVol to be presented as a LUN; this creates LUN 8, using the GUID from the previous system, with block storage as the backing, pointing to the /dev/zvol location.

`root@freebsdhost: ~# ctladm create -b block -l 0 -d 600144F010010600000055577FAA0002 -o file=/dev/zvol/newtank/zvols/esxi-shared`

The above command only creates a temporary LUN presentation via <code>ctl</code>. Upon reboot the ctl config will be forgotten, not the contents of the ZVol. To create a persistent LUN configuration enable
[ctld](https://www.freebsd.org/cgi/man.cgi?query=ctld) via [sysrc](https://www.freebsd.org/cgi/man.cgi?query=sysrc), then create and edit the configuration file, and start the daemon.

`root@freebsdhost: ~# touch /etc/ctl.conf`

`root@freebsdhost: ~# chmod 600 /etc/ctl.conf`

Grab the correct rc.conf parameter to use:

```
root@freebsdhost: ~# service ctld rcvar
# ctld
#
ctld_enable=""
# (default: "")
```

Have sysrc search `rc.conf` and insert or update the flag for ctld_enable:

```
root@freebsdhost: ~# sysrc ctld_enable=YES
ctld_enable: NO -&gt; YES
```

Note the naa. number for each Fiber Channel port:

```
root@freebsdhost: ~# ctladm port -l
Port Online Frontend Name pp vp
0 YES ioctl ioctl 0 0
1 YES tpc tpc 0 0
2 YES camtgt isp0 0 0 naa.2100001b561eeeee
3 YES camtgt isp1 0 0 naa.2100001b561fffff
```

Below creates two Fiber Channel targets with dedicated boot drives and a shared VMFS datastore:

/etc/[ctl.conf](https://www.freebsd.org/cgi/man.cgi?query=ctl.conf&amp;sektion=5)

```
lun esxi-shared-datastore {
 backend block
 device-id 600144F010010600000055577FAA0002
 path /dev/zvol/newtank/zvols/esxi-shared-datastore
}


target naa.2100001b561eeeee {
 alias esxi-host1
 auth-group no-authentication
 port isp0

 lun 0 esxi-shared-datastore

 lun 1 {
 backend block
 device-id 600144F0100106000000000000000009
 path /dev/zvol/newtank/zvols/esxi-os-boot
 }
}

target naa.2100001b561fffff {
 alias esxi-host2
 auth-group no-authentication
 port isp1

 lun 0 esxi-shared-datastore

 lun 2 {
 backend block
 device-id 600144F0100106000011111000000003
 path /dev/zvol/newtank/zvols/esxi2-os-boot
 }
}
```

Start the ctld service:

`root@freebsdhost: ~# service ctld start`

Boot the Fiber Channel initiator and verify everything is working as expected. The drives may need to be reconfigured for the new software provider, but the data will remain. The next in the series will cover importing ESXi virtual machines into bhyve. Very exciting times as [UEFI-GOP is in the works](https://twitter.com/michaeldexter/status/708233317003333632) which effectively would allow a VNC connection to a bhyve vm.

Thanks to [iXsystems](https://www.ixsystems.com) for recently [sponsoring Alexander Motin to improve the code for Qlogic based Fiber Channel
cards](https://www.freebsd.org/news/status/report-2015-10-2015-12.html#Improvements-to-the-QLogic-HBA-Driver). Thanks to Alexander Motin for doing said work.
