---
title: Configure SR-IOV Virtual Functions (VF) in Proxmox LXC containers and VMs
tags: [guide, sriov, virtual-functions, lxc, proxmox, vm]
style: border
color: primary
description: This is a guide to setup Virtual Functions from network adapters capable of SR-IOV in Proxmox on LXC containers and VMs.
---

[Originally posted by me on Reddit](https://www.reddit.com/r/Proxmox/comments/1ak26yg/guide_configure_sriov_virtual_functions_vf_in_lxc/).

# Table of contents

* [Why?](#why)
* [Requirements](#requirements)
* [How to](#how-to)
  * [Enable IOMMU](#enable-iommu)
  * [Configure LXC container](#configure-lxc-container)
    * [LXC Caveats](#lxc-caveats)
  * [Configure VM](#configure-vm)
    * [Attachment](#attachment)
        * [Direct](#direct)
        * [Resource Mapping](#resource-mapping)
    * [VM Caveats](#vm-caveats)
* [Other considerations](#other-considerations)
* [Resources](#resources)
* [Notes](#notes)

# Why?

Using a NIC directly usually yields lower latency, more consistent latency ([stddev](https://en.wikipedia.org/wiki/Standard_deviation)), and offloads the computation work onto a physical switch rather than the CPU when using a Linux bridge ([when `switchdev` is not available](https://www.kernel.org/doc/html/next/networking/switchdev.html)[[1](#note-1)]). CPU load can be a factor for 10G networks, especially if you have an overutilized/underpowered CPU. With SR-IOV, it effectively splits the NIC into sub PCIe interfaces called virtual functions (VF), when supported by the motherboard and NIC. I use Intel's 7xx series NICs which can be configured for up to 64 VFs per port... so plenty of interfaces for my medium sized 3x node cluster.

# Requirements

* Knowledge of networking
* Proxmox
* NIC capable of SR-IOV
* Motherboard that supports SR-IOV
* CPU capable of VT-d

# How to

## Enable IOMMU

This is required for VMs. This is not needed for LXC containers because the kernel is shared.

On EFI booted systems you need to modify `/etc/kernel/cmdline` to include 'intel\_iommu=on iommu=pt' or on AMD systems 'amd\_iommu=on iommu=pt'.

    # cat /etc/kernel/cmdline
    root=ZFS=rpool/ROOT/pve-1 boot=zfs intel_iommu=on iommu=pt
    #

On Grub booted system, you need to append the options to 'GRUB\_CMDLINE\_LINUX\_DEFAULT' within `/etc/default/grub`.

After you modify the appropriate file, update the initramfs (`# update-initramfs -u`) and reboot.

'Boot Mode' will on a Proxmox [node's summary page](https://pve.proxmox.com/pve-docs/pve-admin-guide.html#_nodes) will indicate which boot process is used.

There is a lot more you can tweak with IOMMU which may or may not be required, I suggest checking out the [Proxmox PCI passthrough docs](https://pve.proxmox.com/wiki/PCI_Passthrough).

## Configure LXC container

Create a systemd service to start with the host to configure the VFs (`/etc/systemd/system/sriov-vfs.service`) and enabled it (`# systemctl enable sriov-vfs`). Set the number of VFs to create ('X') for your NIC interface ('&#60;physical-function-nic>'). Configure any options for the VF (see [Resources](#resources) below). Assuming the physical function is connected to a trunk port on your switch; setting a VLAN is helpful and simple at this level rather than within the LXC. Also keep in mind you will need to set 'promisc on' for any trunk ports passed to the LXC. As a pro-tip, I rename the ethernet device to be consistent across nodes with different underlying NICs to allow for LXC migrations between hosts. In this example, I'm appending 'v050' to indicate the VLAN, which I omit for trunk ports.

    [Unit]
    Description=Enable SR-IOV
    Before=network-online.target network-pre.target
    Wants=network-pre.target

    [Service]
    Type=oneshot
    RemainAfterExit=yes

    ################################################################################
    ### LXCs
    # Create NIC VFs and set options
    ExecStart=/usr/bin/bash -c 'echo X > /sys/class/net/<physical-function-nic>/device/sriov_numvfs && sleep 10'
    ExecStart=/usr/bin/bash -c '/usr/bin/ip link set <physical-function-nic> vf 63 vlan 50'
    ExecStart=/usr/bin/bash -c '/usr/bin/ip link set dev <physical-function-nic>v63 name eth1lxc9999v050'
    ExecStart=/usr/bin/bash -c '/usr/bin/ip link set dev eth1lxc9999v050 up'

    [Install]
    WantedBy=multi-user.target

Edit the LXC container configuration (Eg: `/etc/pve/lxc/9999.conf`). The order of the `lxc.net.*` settings is critical, it has to be in the order below. Keep in mind these options are not rendered in the WebUI after manually editing the config.

    lxc.apparmor.profile: unconfined
    lxc.net.1.type: phys
    lxc.net.1.link: eth1lxc9999v050
    lxc.net.1.flags: up
    lxc.net.1.ipv4.address: 10.0.50.100/24
    lxc.net.1.ipv4.gateway: 10.0.50.1

### LXC Caveats

The two caveats to this setup are the 'network-online.service' fails within the container when a Proxmox managed interface is not attached. I leave a bridge tied interface on a dummy VLAN and use a blank static IP assignment which is disconnected. This allows systemd to cleanly start within the LXC container (specifically 'network-online.service' which likely will cascade into other services not starting).

The other caveat is the Proxmox network traffic metrics won't be available (like any PCIe device) for the LXC container but if you have `node_exporter` and Prometheus setup, it is not really a concern.

## Configure VM

Create (or reuse) a systemd service to start with the host to configure the VFs (`/etc/systemd/system/sriov-vfs.service`) and enabled it (`# systemctl enable sriov-vfs`). Set the number of VFs to create ('X') for your NIC interface ('&#60;physical-function-nic>'). Configure any options for the VF (see [Resources](#resources) below). Assuming the physical function is connected to a trunk port on your switch; setting a VLAN is helpful and simple at this level rather than within the VM. Also keep in mind you will need to set 'promisc on' on any trunk ports passed to the VM.

    [Unit]
    Description=Enable SR-IOV
    Before=network-online.target network-pre.target
    Wants=network-pre.target

    [Service]
    Type=oneshot
    RemainAfterExit=yes

    ################################################################################
    ### VMs
    # Create NIC VFs and set options
    ExecStart=/usr/bin/bash -c 'echo X > /sys/class/net/<physical-function-nic>/device/sriov_numvfs && sleep 10'
    ExecStart=/usr/bin/bash -c '/usr/bin/ip link set <physical-function-nic> vf 9 vlan 50'

    [Install]
    WantedBy=multi-user.target

You can quickly get the PCIe id of a virtual function (even if the network driver has been rebinded to `vfio-pci`) by:

    # ls -lah /sys/class/net/<physical-function-nic>/device/virtfn*
    lrwxrwxrwx 1 root root 0 Jan 28 06:28 /sys/class/net/<physical-function-nic>/device/virtfn0 -> ../0000:02:02.0
    lrwxrwxrwx 1 root root 0 Jan 28 06:28 /sys/class/net/<physical-function-nic>/device/virtfn1 -> ../0000:02:02.1
    lrwxrwxrwx 1 root root 0 Jan 28 06:28 /sys/class/net/<physical-function-nic>/device/virtfn2 -> ../0000:02:02.2
    lrwxrwxrwx 1 root root 0 Jan 28 06:28 /sys/class/net/<physical-function-nic>/device/virtfn3 -> ../0000:02:02.3
    lrwxrwxrwx 1 root root 0 Jan 28 06:28 /sys/class/net/<physical-function-nic>/device/virtfn4 -> ../0000:02:02.4
    lrwxrwxrwx 1 root root 0 Jan 28 06:28 /sys/class/net/<physical-function-nic>/device/virtfn5 -> ../0000:02:02.5
    lrwxrwxrwx 1 root root 0 Jan 28 06:28 /sys/class/net/<physical-function-nic>/device/virtfn6 -> ../0000:02:02.6
    lrwxrwxrwx 1 root root 0 Jan 28 06:28 /sys/class/net/<physical-function-nic>/device/virtfn7 -> ../0000:02:02.7
    lrwxrwxrwx 1 root root 0 Jan 28 06:28 /sys/class/net/<physical-function-nic>/device/virtfn8 -> ../0000:02:03.0
    lrwxrwxrwx 1 root root 0 Jan 28 06:28 /sys/class/net/<physical-function-nic>/device/virtfn9 -> ../0000:02:03.1
    ...
    #

### Attachment

There are two options to attach to a VM. You can attach a PCIe device directly to your VM which means it is statically bound to that node OR you can setup a [resource mapping](#resource-mapping) to configure your PCIe device (from the VF) across multiple nodes; thereby allowing stopped migrations of VMs to different nodes without reconfiguring.

#### Direct

Select a VM > 'Hardware' > 'Add' > 'PCI Device' > 'Raw Device' > find the ID from the above output.

#### Resource mapping

Create the [resource mapping](https://pve.proxmox.com/pve-docs/pve-admin-guide.html#resource_mapping) in the Proxmox interface by selecting 'Server View' > 'Datacenter' > 'Resource Mappings' > 'Add'. Then select the 'ID' from the correct virtual function (furthest right column from your output above). I usually set the resource mapping name to the virtual machine and VLAN (eg `router0-v050`). I usually set the description to the VF number. Keep in mind, the resource mapping only attaches the first available PCIe device for a host, if you have multiple devices you want to attach, they MUST be individual maps. After the resource map has been created, you can add other nodes to that mapping by clicking the '+' next to it.

Select a VM > 'Hardware' > 'Add' > 'PCI Device' > 'Mapped Device' > find the resource map you just created.

### VM Caveats

The three caveats to this setup. One, the VM can no longer be migrated while running because of the PCIe device but [resource mapping](#resource-mapping) can make it easier between nodes.

Two, driver support within the guest VM is highly dependent on the guest's OS.

The last caveat is the Proxmox network traffic metrics won't be available (like any PCIe device) for the VM but if you have `node_exporter` and Prometheus setup, it is not really a concern.

# Other considerations

* For my pfSense/OPNsense VMs I like to create a VF for each VLAN and then set the MAC to indicate the VLAN ID (Eg: `xx:xx:xx:yy:00:50` for VLAN 50, where 'xx' is random, and 'yy' indicates my node). This makes it a lot easier to reassign the interfaces if the PCIe attachment order changes (or NICs are upgraded) and you have to reconfigure in the pfSense console.
  * Over the years, I have moved my pfSense configuration file several times between hardware/VM configurations and this is by far the best process I have come up with. I find VLAN VFs simpler than reassigning VLANs within the pfSense console because IIRC you have to recreate the VLAN interfaces and then assign them.
  * Plus VLAN VFs is preferred (rather than within the guest) because if the VM is compromised, you basically have given the attacker full access to your network via a trunk port instead of a subset of VLANs.
* If you are running into issues with SR-IOV and are sure the configuration is correct, I would always suggest starting with upgrading the firmware. The drivers are almost always newer and it is not impossible for the firmware to not understand certain newer commands/features and because bug fixes.
* I also use 'sriov-vfs.service' to set my Proxmox host IP addresses, instead of in `/etc/network/interfaces`. In my `/etc/network/interfaces` I only configure my fallback bridges.

    Excerpt of `sriov-vfs.service`:

        # Set options for PVE VFs
        ExecStart=/usr/bin/bash -c '/usr/bin/ip link set eno1 vf 0 promisc on'
        ExecStart=/usr/bin/bash -c '/usr/bin/ip link set eno1 vf 1 vlan 50'
        ExecStart=/usr/bin/bash -c '/usr/bin/ip link set eno1 vf 2 vlan 60'
        ExecStart=/usr/bin/bash -c '/usr/bin/ip link set eno1 vf 3 vlan 70'
        # Rename PVE VFs
        ExecStart=/usr/bin/bash -c '/usr/bin/ip link set dev eno1v0 name eth0pve0'
        ExecStart=/usr/bin/bash -c '/usr/bin/ip link set dev eno1v1 name eth0pve050' # WebUI and outbound
        ExecStart=/usr/bin/bash -c '/usr/bin/ip link set dev eno1v2 name eth0pve060' # Non-routed cluster/corosync VLAN
        ExecStart=/usr/bin/bash -c '/usr/bin/ip link set dev eno1v3 name eth0pve070' # Non-routed NFS VLAN
        # Set PVE VFs status up
        ExecStart=/usr/bin/bash -c '/usr/bin/ip link set dev eth0pve0 up'
        ExecStart=/usr/bin/bash -c '/usr/bin/ip link set dev eth0pve050 up'
        ExecStart=/usr/bin/bash -c '/usr/bin/ip link set dev eth0pve060 up'
        ExecStart=/usr/bin/bash -c '/usr/bin/ip link set dev eth0pve070 up'
        # Configure PVE IPs on VFs
        ExecStart=/usr/bin/bash -c '/usr/bin/ip address add 10.0.50.100/24 dev eth0pve050'
        ExecStart=/usr/bin/bash -c '/usr/bin/ip address add 10.2.60.100/24 dev eth0pve060'
        ExecStart=/usr/bin/bash -c '/usr/bin/ip address add 10.2.70.100/24 dev eth0pve070'
        # Configure default route
        ExecStart=/usr/bin/bash -c '/usr/bin/ip route add default via 10.0.50.1'

    Entirety of `/etc/network/interfaces`:

        auto lo
        iface lo inet loopback

        iface eth0pve0 inet manual
        auto vmbr0
        iface vmbr0 inet static
        # VM bridge
        bridge-ports eth0pve0
        bridge-stp off
        bridge-fd 0
        bridge-vlan-aware yes
        bridge-vids 50 60 70

        iface eth1pve0 inet manual
        auto vmbr1
        iface vmbr1 inet static
        # LXC bridge
        bridge-ports eth1pve0
        bridge-stp off
        bridge-fd 0
        bridge-vlan-aware yes
        bridge-vids 50 60 70

        source /etc/network/interfaces.d/*

# Resources

* [Intel whitepaper on SR-IOV](https://cdrdv2-public.intel.com/321211/pci-sig-sr-iov-primer-sr-iov-technology-paper.pdf)
   * [Presentation on YouTube](https://www.youtube.com/watch?v=hRHsk8Nycdg)
* [https://pve.proxmox.com/wiki/PCI\_Passthrough](https://pve.proxmox.com/wiki/PCI_Passthrough)
* [Options you can set for a VF - https://manpages.debian.org/unstable/iproute2/ip-link.8.en.html](https://manpages.debian.org/unstable/iproute2/ip-link.8.en.html)
* [https://linuxcontainers.org/lxc/manpages/man5/lxc.container.conf.5.html](https://linuxcontainers.org/lxc/manpages/man5/lxc.container.conf.5.html)

# Notes
<a id="note-1">[1]</a> - I have found only high end NICs support switchdev. For example, I could only find that the Intel 800 series and switch whiteboxes with Mellaxnox chips are capable of using this linux driver.
