---
title: Migrating to FreeBSD - Part 4
tags: [epijunkie.com, series, freebsd, solaris, zfs, stmfadm, fiber channel, lun]
style: border
color: primary
description: Part 4 in a series to migrate to FreeBSD.
comments: true
---

Migrating to FreeBSD from Solaris: Part 4 - Importing ESXi virtual machines to Bhyve
==========================

During [this transition to FreeBSD for my storage host](http://justinholcomb.me/blog/2016/02/28/migration-to-freebsd-part1.html), I wanted to import some ESXi virtual machines formerly hosted on the Solaris
hosted SAN to bhyve. This part of the series will cover everything from powering down a supported VM on ESXi to powering it back on using bhyve.

Caution/Requirement/Notes:

*   Like everything dealing with Windows, it severally bloated this length of this article; my apologizes. A build after commit r288524  (any after 10.3-BETA1+ will work) is required for UEFI booting on which Windows VMs relies.
*   Make a backup of your VMs before attempting this.
*   The ESXi snapshots will need to be deleted for the vmdk file conversion to raw. Without deleting the snapshots, the conversion tools will yield a hard drive image at the state from the first snapshot in the tree.
*   FreeBSD VMs which had snapshots were rendered unbootable after deleting all the ESXi snapshots. In hindsight I should of resisted the urge to use complete (harddrive and RAM) snapshots provided by VMware and used ZFS and [beadm](https://www.freebsd.org/cgi/man.cgi?query=beadm) to take snapshots of the system before upgrades.
*   <strike>There is a tool in ports called [vmdktool,](http://www.freshports.org/sysutils/vmdktool/) it is dependency-free unlike qemu and virtualbox-ose. I found out about vmdktool after writing this article and was not able to fully test it before publishing.</strike> I was not able to get vmdktool to convert my vmdk files (all 5.5+ guests) as it incorrectly complained about "`esxi-guest.vmdk: File too small (must be at least 1024 bytes)`". My guess is the formatting has changed on the newer vmdk formats since the last time this tool was updated.

Steps to take:

1.  Using your preferred method, log onto a VMware management console and power down the intended VM.
2.  Remove the VM from the ESXi inventory to prevent issues.
3.  Copy the entire folder containing the VM to a machine with VirtualBox/QEMU installed. I suggest creating a `jail` with `nullfs` mounts for converting the `vmdk` files without cluttering the base OS install. As mentioned above, the virtual machine must have no snapshots, otherwise using this conversion process will yield a raw image of the first snapshot in the tree, not the disk's current state.
    *   `ssh root@esxihost`
    *   `cd /vmfs/volumes/datastore1`
    *   `scp bguest justin@pcbsdhost:/home/justin/VMs`
4.  In a terminal, navigate to the folder containing the hard drive image files.
    *   `cd /home/justin/VMs/VM1`
5.  Convert the vmdk hard drive image file to a raw image file type. To do this enter the following command:
    *   `VBoxManage clonehd esxi-guest.vmdk image-for-bhyve-volume.raw --format RAW`
    *   or
    *   `qemu-img convert esxi-guest.vmdk -O raw image-for-bhyve-volume.raw`
6.  Grab the file size of the raw image by using `ls -l`.
    *   `ls -l /home/justin/VMs/VM1/raw-image-for-bhyve-volume.raw`
    *   `-rw------- 1 justin justin **107374182400** Feb 27 15:46 /home/justin/VMs/VM1/raw-image-for-bhyve-volume.raw`
7.  Install `iohyve` to manage `bhyve`. [iohyve](https://github.com/pr1ntf/iohyve) is a dependency-free wrapper for `bhyve`, similar to what [iocage](https://github.com/iocage/iocage) is for jails.
    *   Install `iohyve`
        *   `pkg install iohyve`
    *   Configure `tank` as the zpool to use
        *   `iohyve setup pool=tank`
    *   Load kernel modules
        *   `iohyve setup kmod=1`
    *   Setup tap and bridge device auto configure on em0
        *   `iohyve setup net=em0`
    *   Install the `grub-bhyve` binary if using guests other than FreeBSD.
        *   `pkg install grub2-bhyve`
8.  On the Bhyve host, create the bhyve guest in iohyve and set some parameters for the guest.
    *   Create the guest with the exact hard drive size
        *   `iohyve create bguest 107374182400`
        *   `iohyve set bguest ram=2GB`
        *   `iohyve set bguest cpu=2`
    *   If the guest is:
        *   FreeBSD - No further changes needed
        *   Debian 8.x
            *   `iohyve set bguest loader=grub-bhyve`
            *   `iohyve set bguest os=debian8`
        *   Debian 8.x with LVM partition scheme
            *   `iohyve set bguest loader=grub-bhyve`
            *   `iohyve set bguest os=d8lvm`
        *   NetBSD
            *   `iohyve set bguest loader=grub-bhyve`
            *   `iohyve set bguest os=netbsd`
        *   OpenBSD 5.7, 5.8, or 5.9
            *   `iohyve set bguest loader=grub-bhyve`
            *   `iohyve set bguest os=openbsd5x` (Where '`x`' is 7, 8, or 9)
        *   Arch Linux
            *   `iohyve set bguest loader=grub-bhyve`
            *   `iohyve set bguest os=arch`
        *   Windows 7 to 10 and Server 2008 to 2016 (Experimental - Unsupported)
            *   Keep in mind this is highly experimental and success seems to be hit or miss.
            *   Only works for UEFI boot installs. Supposedly you can [convert to UEFI boot](https://www.youtube.com/watch?v=g1eXD30Fox4) process from legacy BIOS booting.
            *   Some prep needs to happen before powering down the Windows guest VM. RDP needs to be enabled and network drivers need to be preinstalled as the serial console for Windows is strictly for diagnostics. With
[work being done](https://twitter.com/michaeldexter/status/708233317003333632) on [UEFI-GOP](https://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface#Graphics_features) for bhyve, a native VNC connection will take the place of needing RDP or the serial interface in the future.
                *   Enable RDP (Windows 2012r2)
                    *   Enter the Windows key
                    *   Type "This PC"
                    *   Right click "This PC"
                    *   Click Properties
                    *   Click "Remote Services"
                    *   Select the radio button "Allow remote connections to this computer"
                    *   Uncheck "Allow connections only from computers running Remote Desktop with Network Level Authentication (recommended)"
                *   [Preinstall VirtIO drivers](https://technet.microsoft.com/en-us/library/cc772036.aspx)
                    *   Download the [VirtIO driver ISO version 0.1.96](https://fedoraproject.org/wiki/Windows_Virtio_Drivers)
                      *   Version later than 0.1.96 do not work.
                    *   Mount the VirtIO driver ISO and extract the contents to the desktop.
                    *   Hit the windows key on the keyboard
                    *   Type "cmd"
                    *   Right click the "Command Prompt" icon
                    *   Click "Run as administrator"
                    *   Enter credentials or click yes.
                    *   In the command shell navigate to the directory containing extracted ISO files:
                        *   `cd C:\Users\Administrator\Desktop\virtio-win-0.1.1\`
                        *   `cd NetKVM\<insert win OS version>\amd64\`
                    *   Stage the VirtIO device drivers with the OS
                        *   `pnputil.exe -a netkvm.inf`
                *   Shutdown the Windows guest in ESXi as described in Step 1.
            *   Back on the FreeBSD bhyve host, download the UEFI userland boot binary firmware.
                *   `wget  https://people.freebsd.org/~grehan/bhyve_uefi/BHYVE_UEFI_20151002.fd`
            *   Install the UEFI firmware as an iohyve resource.
                *   `iohyve cpfw BHYVE_UEFI_20151002.fd`
            *   Create a blank iso image. This is required by the UEFI/Windows loader.
                *   `touch null.iso`
            *   Install the ISO image as a iohyve resource.
                *   `iohyve cpiso null.iso`
            *   Tell iohyve which firmware resource to use when booting from UEFI.
                *   `iohyve set bguest fw=BHYVE_UEFI_20151002.fd`
                *   Required flags. See the [bhyve man page](https://www.freebsd.org/cgi/man.cgi?query=bhyve&apropos=0&sektion=8&manpath=FreeBSD+11-current&arch=default&format=html) for details.
                    *   `iohyve set bguest bargs="-H -w"`
                *   Complete step 9 and return.
                *   Start the VM.
                    *   `iohyve uefi bguest null.iso`
                *   RDP into bguest after a few minutes.
9.  Copy the raw file to the server, if you have not already. Then write the raw image to the ZVol created for `bguest`.
    *   `scp raw-image-for-bhyve-volume.raw root@freebsdserver:/root/`
    *   `dd if=/root/raw-image-for-bhyve-volume.raw of=/dev/zvol/tank/iohyve/bguest/disk0`
10.  Start `bguest` and open the console.
    *   `iohyve start bguest`
    *   `iohyve console bguest`

Seriously, that's it. Be amazed by hardware assisted virtualization happening with tools totaling under 3MB. It is truly remarkable. Big thank-yous go to all the developers who made this possible and also to those who further develop these tools! Also big thanks to Michael Dexter and [pr1ntf](https://twitter.com/pr1ntf).
