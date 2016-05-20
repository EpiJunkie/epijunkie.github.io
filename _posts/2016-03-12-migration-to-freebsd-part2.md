---
layout: post
title: Migrating to FreeBSD - Part 2
date: 2016-03-12
categories: blog
tags:
- data migration
- freebsd
- solaris
- zfs
---

Migrating to FreeBSD from Solaris: Part 2 - Moving data and volumes to new zpool without zfs send and zfs recv
==========================

During <a href="http://epijunkie.github.io/blog/2016/02/28/migration-to-freebsd-part1.html">this transition to FreeBSD for my storage host</a>, the data needed to be moved from one zpool to another zpool which were on 
different zfs versions [5 and 6]. Due to the zfs versions being different, the recommended method of using <code>zfs send|zfs recv</code> is not possible. This creates a large hassle, as <code>zfs send | zfs recv</code> is 
so effective at moving data and keeping the same attributes.

This part of the series will cover moving the files and verifying those files made it safely. Moving volumes will be covered later in the series. This methodology could be adapted for transferring your data from a non-ZFS 
data source to ZFS, however I would recommend using something like md5deep to further <a href="http://blog.epijunkie.com/2010/10/data-verification-after-building-a-raid-5-array-using-freenas/">verify the data made it 
safely</a>.

First, I am going to state the obvious. The fastest way to move bits from one pool to another is directly from one SATA/SAS interface to another. Basically I want to avoid transferring data over a network. In addition to the 
performance benefit of moving the data on a local machine, it is simpler. Commands like <code>find</code> and <code>diff</code> do not work well over a network. Besides physically fitting all the drives, there is 
another interesting problem this presents, how to move the files and volumes from a Solaris zpool to a OpenZFS zpool? My method involves creating a zpool with an older zpool/zfs version that is compatible on both OSes. As 
zpool version 28 and zfs version 5 are the universally compatible version, we will create a pool using those versions. The way to specify that is

```
zpool create -O version=5 -o version=28 oldschool gpt/0001
```

After recreating the dataset structure, the next task is to copy the files over. The best method I have come up with is using <code>rsync</code> with the <code>--remove-source-files</code> flag and this is after taking a 
snapshot of the original pool. I then check to see if all the files were copied over with <code>find</code> and a flag to check for files. Then as one final check I rollback the snapshot and do a <code>diff</code> of the 
directories.

```
rsync -a --remove-source-files /oldtank/ /newtank/
```

Using the find command, you can check to see if any files remain on the old dataset/pool.

```
find /oldtank/ -type f
```

Then I roll back the snapshot and do a `diff` comparison of the old dataset and the new dataset.

```
zfs rollback oldtank@before-rsync
```
```
diff -r /oldtank/ /newtank/ &gt; tank-diff-comparison.log
```

I then do a quick check to see if <code>diff</code> reported any differences.

```
cat tank-diff-comparison.log | grep "Only in"
```

If anything comes up, I open the log file to check specifically what the issue was. Most of the time it is .DS_Store files which I ignore.

Some thing to keep in mind is the expansion of size on compressed pools. You might be very surprised and when your data no longer fits if you are close to your pool capacity or have a good COMPRESSRATIO.

Another thing to keep in mind is if you create additional datasets on Solaris it will make those dataset inaccessible on an OpenZFS platform. Apparently it will mark the dataset as a version 6 file system. I am glad this 
happened on one of my smaller dataset, as I had to start the process over. The exact message is:

```
root@host: ~# zfs mount tank/dataset cannot mount 'tank/dataset': Can't mount a version 6 file system on a version 28 pool. Pool must be upgraded to mount this file system.
```

That is about it for this part of the series. This is a short explanation but it will take hours if not days to complete depending on the size of your pool.
