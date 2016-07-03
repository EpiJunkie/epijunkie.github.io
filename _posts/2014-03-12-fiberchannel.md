---
layout: post
title: Fiber Channel for a homelab
date: 2014-03-12
tags: bhyve poudriere
category: blog
---

# Fiber Channel for a homelab

As I have learned more advance features of VMware's virtualization products, my rack has slowly been filling up with servers for the clustered based technologies. With the increase of servers there has been a steady increase in the amount of traffic to my SAN server as well. I was utilizing iSCSI for my SAN and while it works pretty well, the traffic is on a shared medium (Ethernet) so there was some delay from other traffic in the same broadcast domain. Especially when there was a high network load.

While I could have placed the iSCSI traffic on a dedicated VLAN and/or implemented QoS to gained a performance bump at no cost. I decided to implement Fiber Channel rather than other options. Like iSCSI, Fiber Channel is a storage oriented protocol. Unlike iSCSI, Fiber Channel can only be on a dedicated media, meaning the only traffic transversing the optical fiber is the Fiber Channel Protocol. One of the many benefits of Fiber Channel is that it can be setup at a low cost; currently (July 3, 2016) $31 for the host and first client, and +$9 for each additional client, for up to 4 total clients and one host. After four clients have been reached, a Fiber Channel switch or an additional 4x port card is required. What is impressive is at those prices you are getting an optical based media using a storage oriented protocol that operates at 4.25Gb/s. These Qlogic Fiber Channel PCIe card adapters have drivers for FreeBSD, Windows, Apple's OS X, Red Hat, SUSE, Solaris, VMware ESXi, and Xen. The cards support boot-from-Fiber-Channel which means running the servers completely diskless is possible. Further more, even though a switch is not in use, the VMFS datastore can be shared between the other FC clients.

A typical hard drives will only push between 100-145MB/s, so you may wonder why I would want a 4.25Gb/s network for my SAN traffic when the hard drive's max through-put is mostly within the operating speed of a gigabit Ethernet network. Well, with FreeBSD the ARC and L2ARC should be able to provide burst speeds well beyond what GbE is capable of.

Creating a 10Gbe Ethernet network was also an option but the cost is still incredibly high. The price of a 10Gbe switches is still in the thousands for anything more than two 10GbE ports. This high cost of 10Gbe Ethernet is ultimately why I selected Fiber Channel over 10Gbe Ethernet. For the cost of a point-to-point Fiber Channel infrastructure, I could only purchase a single 10GbE NIC.

Fiber Channel is a pretty cool technology and was challenging to initially setup and understand with the limited resources outside of an enterprise support contract. It is important to note that I barely have touched the surface of this cool technology, there are so many more features are available such as bonding, arbitrated loops, Fiber Channel over Ethernet that I haven't even touched in this article. I encourage you to look into it.

&nbsp;

Price/Parts Breakdown (Prices updated July 3, 2016):

Qlogic QLE2464 - 4x port PCIe adapter - <a href="http://www.ebay.com/sch/i.html?_sacat=0&amp;_from=R40&amp;_nkw=QLE2464&amp;_sop=15">$22</a>

Qlogic QLE2460 - 1x port PCIe adapter - <a href="http://www.ebay.com/sch/i.html?_odkw=QLE2464&amp;_sop=15&amp;_osacat=0&amp;_from=R40&amp;_nkw=QLE2460&amp;_sacat=0">$5</a>

LC to LC OM3 Fiber Cable - <a href="http://www.ebay.com/sch/i.html?_from=R40&amp;_sacat=0&amp;_nkw=fiber+om3+LC&amp;_sop=15">$4</a>
