---
title: ZFS Volume Manipulation
tags: [freebsd, solaris, zfs, zvols]
style: border
color: primary
description: Ramblings about ZFS and zvols.
comments: true
---

ZFS volume manipulations and best practices
==========================

I am a strong advocate of ZFS. I have been using ZFS for six years now and it has been a wonderful experience. In that time I have primarily installed consumer harddrives and SSDs, some of which ran bad firmware or poorly written HBA drivers. Many drives have failed, including one tragic night when three drives [gave up the ghost](https://www.google.com/search?q=give+up+the+ghost+idiom).

Surprisingly I have had but one incident of data loss during that time which I am chalking up to my own ignorance of [pool recovery](https://docs.oracle.com/cd/E53394_01/html/E54801/gavwg.html) and a misconfigured UPS shutdown command. In short that incident was brought on by a power failure and a host causing a lock on a block storage ZVol which in turn caused a kernel panic upon importing the pool. In hindsight I probably could of recovered the data had I mounted the pool in read-only mode and `dd`-ed the data to a file or [rolled back on the transaction group](https://www.reddit.com/r/zfs/comments/478wwd/lost_power_during_a_zfs_receive_and_now_cant/).

Recently I have been restructuring my pools and building a machine for a full offsite replication of my data. During the testing of the backup, it has involved moving the ZVols which is not as thoroughly documented portion in the FreeBSD man pages.

The point of this article is to describe some of the gotchas with managing ZVols, in the hopes that it can spare you living the same pain points.

Setting a user defined zfs property for the LUN value
-----
I highly recommend to set a [user defined property](https://docs.oracle.com/cd/E19120-01/open.solaris/817-2271/gdrcw/index.html) on each iSCSI/FC ZVol dataset to keep track of the LU device-id associated with it. The reason is, the device-id is not kept with the ZVol but in fact kept on the OS drive. This can be a problem if you lose your OS drive or move the ZVol to another pool in a `zfs send | zfs recv` operation, and then want to import the LU from that new location to the same hosts. For example, ESXi will not recognize the same block storage device under a new device-id number.

First grab the LU device-id value:

```
root@solaris:~# stmfadm list-lu -v | egrep "LU|Data File"
LU Name: 600144F010010600000065677FFF1234
Data File : /dev/zvol/rdsk/tank/zvols/esxi-shared-storage
```

or

```
root@freebsd:~ # ctladm devlist
LUN Backend       Size (Blocks)   BS Serial Number    Device ID       
  0 block         401152851968                        600144F010010600000065677FFF1234
```

Then set the user define property:

```
epijunkie@anyOS:~# zfs set lu:device-id=600144F010010600000065677FFF1234 tank/zvols/esxi-shared-storage
```

Command explanation:

*   `zfs set` - this tells ZFS you are going to set a property
*   `lu:` - the ":" tells ZFS it is an arbitrary user define property with a name of "lu". Think of this as a group or application level name.
*   `device-id` tells ZFS what the arbitrary property name.
*   `=600144F010010600000065677FFF1234` tells ZFS to assign a value of "600144F010010600000065677FFF1234" to the user define property
*   `tank/zvols/esxi-shared-storage` is the zvol dataset name.

Check to see if the property is set:

```
epijunkie@anyOS:~# zfs get all tank/zvols/esxi-shared-storage
NAME                             PROPERTY              VALUE                             SOURCE
...<REDACTED>...
tank/zvols/esxi-shared-datastore lu:device-id          600144F010010600000065677FFF1234  local
```

The last line shows the new property, this property will transpire with a `zfs send | zfs recv` transaction when the `-R` flag is used on the `zfs send` side.

Snapshots
----
This was really a kick-in-the-ass-by-reality when I issued this command and noticed my ZVol had drastically increased the referred data. When taking a snapshot of a ZVol, the volume must _be able_ to change every bit that is reserved to it. Which means if your ZVol is created with 5GB of space and has 2GB written to it, when you create a snapshot the ZVol will now be consuming 7GB of space. The same is true for setting `reservations` and `refreservations`.

This seems obvious now now that it is pointed out but it is a hard reality when trying to take a snapshot of a ZVol and are unable to. I was made aware of this behavior when attempting to snapshot a ZVol on a flash based pool which consumed most of the pool capacity. Which brings us to the next topic.

Moving a ZVol using `dd`
----
Typically when you want to move a ZVol from one pool to another, the best method is using `zfs send | zfs recv`. However there are at least two scenarios when this would not be possible: when moving a ZVol from a Solaris pool to a OpenZFS pool or when taking a snapshot is not possible such as the case when there are space constrains. While a `zfs send | zfs recv` can be done across pools of different `zpool` versions [1-37;5000], it can not be done across `zfs` versions [1-6].  This is a problem when sending a dataset from Solaris `zfs` version 6 and receiving on a OpenZFS based system which is on version 5.

In this situation because a snapshot can not be created, it is recommended to turn off all services that can modify the ZVol as having data at different states will surely cause issues.

```
root@solaris:~# svcadm disable stmf
```

or

```
root@freebsd:~ # service ctld stop
```

Next lets write the ZVol contents to a gzipped file for transportation.

```
epijunkie@oldhost:~# dd if=/dev/zvol/rdsk/tank/zvols/esxi-shared-storage | gzip > esxi-shared-storage.zvol.img.raw.gz
```

The volume sizes must match or the destination needs to be larger, not because ZFS or `dd` but that the host being served the block device will likely break especially when it attempts to write data to sector of the disk it thought existed. Take note that creating a 3T ZVol on Solaris is not the same size as 3T on FreeBSD. Using the `-p` flag will display the exact values, not the human friendly equivalents.

```
root@oldhost:~# zfs get -p volsize tank/zvols/esxi-shared-storage
NAME                               PROPERTY VALUE        SOURCE
tank/zvols/esxi-shared-datastore   volsize  401152851968 local
```

On the destination machine, first create a ZVol.

```
root@newhost:~# zfs create -V 401152851968 NEWtank/zvols/esxi-shared-storage
```

Un-gzip the file and `dd` to the ZVol.

```
root@newhost:~# dd if=esxi-shared-storage.zvol.img.raw.gz | gunzip | dd of=/dev/zvol/rdsk/NEWtank/zvols/esxi-shared-storage
```

Next re-create the LU using the same number as before.

```
root@newsolaris:~# stmfadm create-lu --lu-prop guid=600144F010010600000065677FFF1234 /dev/zvol/rdsk/NEWtank/zvols/esxi-shared-storage
```

or

```
root@newfreebsd:~ # ctladm create -b block -l 0 -d 600144F010010600000065677FFF1234 -o file=/dev/zvol/rdsk/NEWtank/zvols/esxi-shared-storage
```

Mounting a zpool contained on a ZVol.
----
Recently I had an experience where a misconfiguration on a virtual FreeBSD machine rendered a non-bootable environment. Fortunately this machine was actually on the to-do list to migrate to a jail, so this expedited the process. Recently this vm was converted over to `bhyve` so I imported the pool and copied over the needed data. Importing a zpool from a ZVol is not straight forward, even pointing the import command to the directory does not yield any results by default. This is where changing the `volmode` comes into play. There are 4 user settable modes to place a volume in as follows:

*   `default` which is what is set by the`sysctl` "`vfs.zfs.vol.mode`" on FreeBSD machines. This typically is set to `geom` mode, however I was having issues with a particular volume which required expressly setting the mode to `geom` even though the default was already set to geom.
*   `geom` or mode _1_ - "exposes volumes as geom(4) providers, providing maximal functionality."[1]
*   `dev` or mode _2_ -  "exposes volumes only as cdev device in devfs. Such volumes can be accessed only as raw disk device files, i.e. they can not be partitioned, mounted, participate in RAIDs, etc, but they are faster, and in some use scenarios with untrusted consumer, such as NAS or VM storage, can be more safe."[1]
*   `none` or mode _3_ -  "are not exposed outside ZFS, but can be snapshoted, cloned, replicated, etc, that can be suitable for backup purposes."[1] This is also a helpful mode to set on a bhyve template as accidental modifications are harder to achieve.

As mentioned before, this particular vm was troublesome and did not import using `zpool import -d /dev/zvol/tank/vm-zvol -f -R /mnt/rpool rpool` command as it should have. Not wanting to reboot the system and being okay with the changes not saving to the ZVol, I opted to clone the ZVol after explicitly setting the `volmode` to `geom`.

```
root@host:~# zfs set volmode=geom tank/vm-zvol
root@host:~# zfs snapshot tank/vm-zvol@troublesome
root@host:~# zfs clone tank/vm-zvol@troublesome tank/vm-zvol-clone
```

Now `zpool import` will recognizes the ZVol as an available pool to import. Import it and mount it to a temporary location of `/mnt/temp-vm` as not to clobber the existing root.
```
root@host:~# zpool import -d /dev/zvol/tank/vm-zvol-clone -f -R /mnt/temp-vm tank
```
Conclusion
----
 With the development of `byhve` which can use ZVols as block storage backing for the virtual machines, I suspect ZVol manipulations will become more common among users. Hopefully this will spare you some pain points if not at least make you aware of them.

 References
 ----
 [1] - [FreeBSD man pages - zfs(8)](https://www.freebsd.org/cgi/man.cgi?zfs(8))
