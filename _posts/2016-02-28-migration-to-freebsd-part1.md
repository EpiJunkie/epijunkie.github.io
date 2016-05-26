---
layout: post
title: Migrating to FreeBSD - Part 1
date: 2016-02-28
tags: freebsd solaris zfs
category: blog
---

Migrating to FreeBSD from Solaris 11: Part 1 - About
==========================

Where am I coming from?
----
I am currently in the process of switching from a Solaris 11 storage system to one running FreeBSD. I have used Solaris 11 as my primary storage appliance since 2010\. In 2010 FreeNAS was still UFS based and ZFS support was considered experimental. Also IIRC, FreeBSD's support of ZFS had just been released. My lack of experience/knowledge, aversion to risk of data, and the lure of native encryption influenced my decision to use Solaris.

Why am I switching?
----
In my not so humble opinion, I believe FreeBSD is the best platform for nearly all computing needs. The factual reasons include the stability, the community, the BSD license, the built-in type-2 hypervisor called [bhyve](http://www.bhyve.org/), the solid ZFS implementation, the recent packages/ports, the whole system approach, and the methodical-forward-thinking development process make FreeBSD one, if not the best platform for computing.

I mentioned encryption.
----
My primary use for encryption is that it makes warranty returns on hard drives "[No big deal](https://youtu.be/urzWU9gDKfc?t=20s)" as Leslie Chow put it. You see, I buy cheap drives intended for light desktop use but then abuse them with 24/7 SAN & NAS use. I know I am a terrible person for the flagrant abuse of hard drives. That said, GELI will be used in place of the native encryption provided in the Solaris-fork.

What services am I replacing?
----
Currently files are shared over SMB and ZVols are served over Fiber Channel. However to make this project even more complex and challenging, this machine's duties will expand to provide more services. The primary reason for adding services to this machine is to consolidate the always-on hardware to one server and one DAS.

Goals
----
*   Move core services currently running as ESXi virtual machines to either jails or bhyve VMs. The goal is to leave the ESXi and Hyper-V hosts strictly for lab use.
*   Minimize services running directly on this storage host.
*   Continue to share files via Samba.
*   Continue to share block storage devices (ZVols) through Fiber Channel on Qlogic cards.
*   Serve as backup target for FreeBSD desktops and laptop through `zfs send|zfs recv`.
*   Initiate backups to offsite server via incremental `zfs send|zfs recv`.
*   Continue to use encryption on drives for hard drive warranty replacement ease and cost.
*   Possibly the most interesting goal of this transition, is to be able to quickly transition a bhyve vm's volume to being served via Fiber Channel on bare metal.
    *   My use case is a `poudriere` build machine. The work flow would be something like this:
        1.  Setup the `poudriere` build host in a bhyve vm.
        2.  Power off bhyve vm and reconfigure to be served via Fiber Channel.
        3.  Run volume as bare metal over FC on a faster and more powerful server to build the packages.
        4.  Then switch back to running as a bhyve vm to serve the packages.

What specific 'core' services will be hosted as virtual machines?
----
*   pfSense #1 - bhyve - Primary gateway and firewall for the network. This will route traffic between 11 VLANs.
*   pfSense #2 - bhyve - Incoming VPN traffic.
*   pfSense #3 - bhyve - Outgoing VPN traffic.
*   FreeBSD #1 - jail - Internal website for lab wiki and private journal via `nginx` and `MariaDB`.
*   FreeBSD #2 - jail - Primary DNS resolution and ad blocking through `unbound` and a clever `nginx` configuration.
*   FreeBSD #3 - bhyve and Fiber Channel - `poudriere` build machine.
*   FreeBSD #4 - bhyve - Internal root certificate authority.
*   FreeBSD #5 - bhyve - Internal intermediate certificate authority.
*   FreeBSD #6 - jail - Serve music through Plex using `nullfs` to pull music from the host.
*   FreeBSD #7 - jail - `postfix`
*   FreeBSD #8 - jail - GPS time clock
*   FreeBSD #9 - Monitoring of host, still researching a solution. A jail would be preferred.
*   Debian #1 - bhyve - [Ubiquiti mFi](https://www.youtube.com/watch?v=wxJ_mKO3eRg) power usage logging appliance.

The above services and VMs seems like a lot to ask a single machine to do, however the host has 192GB of RAM and the data is stored on eight pairs of striped mirrors.

Anticipated pain points and what I plan to document in this series:
----
*   [Moving files and volumes from Solaris zpool without `zfs send|zfs recv` due to the zpool version incompatibility, including how to verify all the data was transferred over](http://justinholcomb.me/blog/2016/03/12/migration-to-freebsd-part2.html).
*   [Getting FiberChannel target mode working with Qlogic cards](http://justinholcomb.me/blog/2016/03/19/migration-to-freebsd-part3.html).
*   [Moving current ESXi virtual machines to bhyve](http://justinholcomb.me/blog/2016/03/26/migration-to-freebsd-part4.html).
*   [Using jails, what is gained and what is lost by using them](http://justinholcomb.me/blog/2016/04/03/migration-to-freebsd-part5.html).
*   ZFS replication as a target (`zfs recv`) and as an initiator (`zfs send`).

This series is being written to document the process to transition a large ZFS pool from one system to another, this includes the services which depend on that data, in the hopes that the series will inspire, inform, or help in some way.
