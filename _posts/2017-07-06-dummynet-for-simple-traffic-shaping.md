---
layout: post
title: Using dummynet for simple bandwidth control
date: 2017-07-06
tags: freebsd dummynet bandwidth control
category: blog
---

Using dummynet for simple bandwidth control
==========================

Recently a project came up at work that required limiting the fast WAN connection to our building, to a much slower rate for development. More specifically, the purpose was to test edge cases for bandwidth/latency sensitive workloads. A proof of concept was built on an Ubuntu virtual machine to demonstrated that both latency and bandwidth could be manipulated using [OSS](https://en.wikipedia.org/wiki/Open-source_software) rather than some [crazy expensive commercial appliance](http://www.kernelsoftware.com/products/catalog/apposite.html). Ubuntu was not my first choice, so FreeBSD was mentioned as an (better) alternative. Thankfully, FreeBSD was an amenable solution.

While normally deploying something like this would be done on an old-previously-discarded PC, for this project a smaller footprint was desired. An embedded device from [PC Engine](https://pcengines.ch/about.htm) was selected, specifically their [APU2C4](https://pcengines.ch/apu2c4.htm).

This article will go over installing FreeBSD 11.0-RELEASE on a PC Engine APU2C4, then configuring it to run inline (bridge-mode) to limit bandwidth for all the downstream devices.


Required components
-----

- A copy of [CHECKSUM.SHA512-FreeBSD-11.0-RELEASE-amd64](https://download.freebsd.org/ftp/releases/amd64/amd64/ISO-IMAGES/11.0/CHECKSUM.SHA512-FreeBSD-11.0-RELEASE-amd64)
- A copy of [FreeBSD-11.0-RELEASE-amd64-memstick.img.xz ](https://download.freebsd.org/ftp/releases/amd64/amd64/ISO-IMAGES/11.0/FreeBSD-11.0-RELEASE-amd64-memstick.img.xz)
- Spare 1GB or larger USB thumbdrive
- PC Engine's [APU2C4](https://pcengines.ch/apu2c4.htm)
- PC Engine's [APU enclosure](https://pcengines.ch/case1d2u.htm) (acts as a heatsink and **IS** required unless a custom case is built with a 5mm standoff)
- PC Engine's [APU Power Supply](https://pcengines.ch/ac12vus2.htm)
- mSATA SSD. This guide used [this 16GB one](https://pcengines.ch/msata16g.htm).
- Serial port on your PC or a [USB serial adapter](https://www.amazon.com/dp/B00425S1H8/).
- Null modem cable or [adapter](https://www.amazon.com/dp/B000DZH4V0/).

We recommend buying the [APU2C4 kit](https://embeddor.com/product/apu-kit/#board) from [Embeddor](https://embeddor.com). They were very quick to ship the device and helpful in giving advice for this project.


Assemble the APU2C4
-----
This part of the article defers to [the factory documentation](http://pcengines.ch/apucool.htm) to assemble the APU2C4. Also install any miniPCI cards that you will be using, including the mSATA storage device. Keep in mind that FreeBSD 11.0 does not have device drivers written for the SD card reader found on the APU2C4.


Write the FreeBSD memstick image to a USB
-----
First download the [list of known hashes](https://download.freebsd.org/ftp/releases/amd64/amd64/ISO-IMAGES/11.0/CHECKSUM.SHA512-FreeBSD-11.0-RELEASE-amd64) for the 11.0-RELEASE.

Next [download the xz-ed -memstick.img image](https://download.freebsd.org/ftp/releases/amd64/amd64/ISO-IMAGES/11.0/FreeBSD-11.0-RELEASE-amd64-memstick.img.xz) to save the FreeBSD Foundation some bandwidth costs.

Now compare the SHA512 hash of the image you just downloaded with the expected hash. On OSX, the following command would be used:

    $ shasum -a 512 FreeBSD-11.0-RELEASE-amd64-memstick.img.xz
    4cf01fc51d9f89bc581262525ebb30299443c3b86b309cc8230b6eed778afcb4776a6d602dcf85b2bbe1fde824c2cda8cbeed8ab57bb03103e369ca73880525b  FreeBSD-11.0-RELEASE-amd64-memstick.img.xz

After the hash has been verified, write the image to a spare USB thumbdrive. Use `dd`, which is colloquially known as 'disk destroyer', so be sure that `/dev/rdisk9` is the correct disk in your environment before writing the image.

    # gunzip --stdout FreeBSD-11.0-RELEASE-amd64-memstick.img.xz | dd of=/dev/rdisk9 bs=1m

See the FreeBSD handbook for complete details on [writing an image file to USB](https://www.freebsd.org/doc/handbook/bsdinstall-pre.html#bsdinstall-usb) if needed.


Prepare the installation media for booting on the APU2C4
-----
Boot the imaged USB from a regular PC that has a VGA console. A few changes need to be made to allow it to boot from serial on the APU2C4.

The first prompt presented when booting from the imaged USB is the welcome screen asking whether to start the installation or drop into a shell. Select "Shell".

![Welcome - FreeBSD 11.0 Installer](/assets/img/blog/201707-bwctl/00-drop-to-shell.png)

Remount the root partition as read-write by using the following command:

    mount -rw /

Append to `/boot/loader.conf` using your favorite text editor:

    boot_multicons="YES"
    boot_serial="YES"
    comconsole_speed="115200"
    console="comconsole,vidconsole"

After the above modifications have been made, the APU2C4 will successfully boot the installation media with a serial console.


Install FreeBSD to the APU2C4
----
First connect your PC to the APU2C4 via a serial port using a null modem adapter/cable. Then connect to the serial console:

    $ sudo cu -l /dev/cu.usbserial -s 115200
    Password:
    Connected.

Plug-in the imaged USB thumbdrive and then the power adapter into the APU2C4 to start the boot process.

    PCEngines apu2
    coreboot build 20170228
    4080 MB ECC DRAM

    SeaBIOS (version rel-1.10.0.1)

    Press F10 key now for boot menu

Press F10 to ensure you select the correct device to boot from.

    Select boot device:
    --
    1. USB MSC Drive Sandisk USB Ultra 1100
    2. ata0-0: SATA SSD ATA-11 Hard-Disk (15272 MiBytes)
    3. Payload [memtest]
    4. Payload [setup]

    Booting from Hard Disk...
    gptboot: backup GPT header checksum mismatch

After selecting the USB with the memstick installation and a few moments, you will be prompted with the following screen:

    Welcome to FreeBSD!

    Please choose the appropriate terminal type for your system.
    Common console types are:
       ansi     Standard ANSI terminal
       vt100    VT100 or compatible terminal
       xterm    xterm terminal emulator (or compatible)
       cons25w  cons25w terminal

    Console type [vt100]: xterm

Type in "xterm" for the best experience. Select "Install" at the FreeBSD installer welcome screen.

![Welcome - FreeBSD 11.0 Installer](/assets/img/blog/201707-bwctl/01-first-screen.png)

Select the appropriate keymap for your system.

![Keymap Selection - FreeBSD 11.0 Installer](/assets/img/blog/201707-bwctl/02-keymap.png)

Input a hostname.

![Set Hostname - FreeBSD 11.0 Installer](/assets/img/blog/201707-bwctl/03-hostname.png)

Deselect all system components for installation.

![Distribution Select - FreeBSD 11.0 Installer](/assets/img/blog/201707-bwctl/04-system-components.png)

Select "Auto (ZFS)" option as this system will likely never be powered off correctly and the atomic writes that come with ZFS will yield better results.

![Partitioning - FreeBSD 11.0 Installer](/assets/img/blog/201707-bwctl/05-filesystem-select.png)

The defaults options on this screen are sufficient. Hit enter when ready to continue to the next screen.

![ZFS Configuration - FreeBSD 11.0 Installer](/assets/img/blog/201707-bwctl/06-zfs-conf.png)

Select "stripe" unless you have two mSATA devices that will be mirrored.

![ZFS Configuration - Select vdev type - FreeBSD 11.0 Installer](/assets/img/blog/201707-bwctl/07-zfs-conf-vdev.png)

Select "ada0" for the mSATA device.

![ZFS Configuration - Select Disks - FreeBSD 11.0 Installer](/assets/img/blog/201707-bwctl/08-zfs-disk-selection.png)

Select "Yes" only after you are sure this device contains no data, all contents should be considered lost after this step.

![ZFS Configuration - Confirm Destroy - FreeBSD 11.0 Installer](/assets/img/blog/201707-bwctl/09-zfs-disk-confirm-destroy.png)

A couple of screens will flash by with the installation progress.

![Install Progress - FreeBSD 11.0 Installer](/assets/img/blog/201707-bwctl/10-install-progress2.png)

After a few moments you will be prompted for a root password.

    FreeBSD 11.0 Installer
    ========================

    Please select a password for the system management account (root):
    Changing local password for root
    New Password:

There are no restrictions or requirements on this password. Then select the first Intel NIC and select "OK".

![Network Configuration - NIC Selection - FreeBSD 11.0 Installer](/assets/img/blog/201707-bwctl/11-net-nic-selection.png)

Select "No" as we will be configuring this device in `/etc/rc.conf` after the installation.

![Network Configuration - IPv4 Configuration - FreeBSD 11.0 Installer](/assets/img/blog/201707-bwctl/12-net-ipv4.png)

Select "No" for the IPv6 as well.

![Network Configuration - IPv6 Configuration - FreeBSD 11.0 Installer](/assets/img/blog/201707-bwctl/13-net-ipv6.png)

Select "No" for the UTC CMOS prompt.

![Time Zone Selector - UTC Time on CMOS - FreeBSD 11.0 Installer](/assets/img/blog/201707-bwctl/14-utc-cmos.png)

Select the appropriate region.

![Time Zone Selector - Select Region - FreeBSD 11.0 Installer](/assets/img/blog/201707-bwctl/15-time-region.png)

Select the appropriate country

![Time Zone Selector - Select  - FreeBSD 11.0 Installer](/assets/img/blog/201707-bwctl/16-time-country.png)

Select the appropriate time zone

![Time Zone Selector - Select  - FreeBSD 11.0 Installer](/assets/img/blog/201707-bwctl/17-time-zone.png)

Confirm the timezone and select "Yes".

![Time Zone Selector - Confirm Selection - FreeBSD 11.0 Installer](/assets/img/blog/201707-bwctl/18-time-confirm.png)

Select "Skip" when configuring the calendar.

![Time & Date - Calendar - FreeBSD 11.0 Installer](/assets/img/blog/201707-bwctl/19-set-calendar.png)

Select "Skip" when setting the time.

![Time & Date - Time - FreeBSD 11.0 Installer](/assets/img/blog/201707-bwctl/20-set-time.png)

In addition to the default services, also select "ntpd" and "powerd".

![System Configuration - FreeBSD 11.0 Installer](/assets/img/blog/201707-bwctl/21-select-services.png)

Select "Disable Sendmail service".

![System Hardening - FreeBSD 11.0 Installer](/assets/img/blog/201707-bwctl/22-hardening.png)

For brevity of this guide, we will select "No" when prompted to create a user.

![Add User Accounts - FreeBSD 11.0 Installer](/assets/img/blog/201707-bwctl/23-user-creation.png)

At the final configuration, select "Exit".

![Final Configuration - FreeBSD 11.0 Installer](/assets/img/blog/201707-bwctl/24-final-conf.png)

Select "Yes" to make some final modifications from a shell.

![Manual Configuration - FreeBSD 11.0 Installer](/assets/img/blog/201707-bwctl/25-manual-conf.png)

We will continue from this prompt in the next section.

    This shell is operating in a chroot in the new system. When finished making configuration changes, type "exit".
    #

DO NOT reboot just yet.


Post installation changes
----
This section continues from the shell prompt after the installation in the previous section but before rebooting. Some changes need to be made for a successful first boot and we will also configure `dummynet` in `ipfw`.

Append to `/boot/loader.conf` with your favorite text editor and input the following:

    boot_multicons="YES"
    boot_serial="YES"
    comconsole_speed="115200"
    console="comconsole,vidconsole"
    dummynet_load="YES"
    if_bridge_load="YES"

The first four entries are the same modifications made to the installation media earlier and are still necessary to view the console via serial on the APU2C4. The `dummynet_load="YES"` entry will load the `dummynet` kernel module on boot and `if_bridge_load="YES"` will load the `if_bridge` kernel module on boot.

Run the following commands to update `/etc/rc.conf`:

    sysrc firewall_enable="YES"
    sysrc firewall_script="/etc/firewall"
    sysrc cloned_interfaces="bridge0"
    sysrc ifconfig_bridge0="addm igb0 addm igb2"
    sysrc ifconfig_igb0="up"
    sysrc ifconfig_igb2="up"

The first entry `firewall_enable="YES"` enables the `ipfw` firewall using the script located at the next entry (`firewall_script="/etc/firewall"`). Later we will create this file (`/etc/firewall`) and give it execute permissions. The next entry, `cloned_interfaces="bridge0"`, creates a `bridge0` device which will allow traffic to bridge from the inlet interface (`igb0`) to the outlet interface (`igb2`). The next entry `ifconfig_bridge0="addm igb0 addm igb2"`, adds both interfaces to the bridge. The next and last two entries put the interfaces in the "up" configuration to allow traffic to pass, this is required because no IP address is being configured on these interfaces.

Append to `/etc/sysctl.conf` with your favorite text editor and input the following:

    net.link.bridge.ipfw=1
    net.inet.ip.fw.one_pass=0

The first entry is required so that `ipfw` can process traffic that passes through bridges. The second entry allows for `ipfw` to continue processing a packet after being passed through a pipe.

We will continue from this shell in the next section. DO NOT reboot just yet.


Basic `ipfw` configuration for bandwidth restriction
----

Create a file called `/etc/firewall` and give it execution permission

    touch /etc/firewall
    chmod +x /etc/firewall

Edit `/etc/firewall` with your favorite text editor and input the following:

    #!/bin/sh

    # Removes all previous rules and pipe configurations.
    ipfw -q flush
    ipfw -q pipe flush

    # Downstream limit
    ipfw pipe 1 config bw 7mbits/s
    ipfw add pipe 1 ip from any to any out recv igb0

    # Upstream limit
    ipfw pipe 2 config bw 896kbits/s
    ipfw add pipe 2 ip from any to any out recv igb2

    # Needed to pass traffic after being processes by dummynet/pipe.
    ipfw add 10000 allow ip from any to any

That is it. After saving this file and rebooting, the device is ready to be deployed on the network.


Further modifications
----
Here are further configuration options to consider:

- Command to add a 100ms delay to traffic:

      ipfw pipe 1 config delay 100ms

- Command to loose 10% of packets:

      ipfw pipe 1 config plr 0.1

- Command to limit bandwidth to 1000Kbit/s, add a 100ms delay, and loose 10% of packets:

      ipfw pipe 1 config bw 1000kbits/s delay 100ms plr 0.1

- Access the device via SSH.

  The way the embedded device is configured above, it is transparent and management requires a serial console. However, you may want to temporarily have network access by grabbing an IP from DHCP:

      dhclient igb0

  Or persistently grab an IP by running the following command to modify the entry in `/etc/rc.conf`:

      sysrc ifconfig_igb0="DHCP"

- Use the third interface (`igb1`) for traffic capture.

  This adds `igb1` as a span device which copies all traffic on the bridge to that interface, the next command enters an entry in `/etc/rc.conf` to bring "up" the device on boot.

      sysrc ifconfig_bridge0="addm igb0 addm igb2 span igb1"
      sysrc ifconfig_igb1="up"

- Limit individual hosts with separate unshared limits.

  So with the above basic configuration, all hosts share the same 7Mb/896Kb pipe. However if you want to have different rates simultaneous applied to different hosts this is possible by using additional pipes. It is also possible to have dedicated pipes so each host get's it's own 7Mb/896Kb "connection". This should scale to the limits of the ISP's service bandwidth or to the CPU's limits, whichever is lower and likely going to be the ISP's service bandwidth.

      #!/bin/sh

      # Removes all previous rules and pipe configurations.
      ipfw -q flush
      ipfw -q pipe flush


      # Create a group of IPs to restrict
      host_group_2="{ 192.168.0.102 or 192.168.0.103 or 192.168.0.104 }"

      # Host #1
      # Downstream
      ipfw pipe 1 config bw 7mbits/s
      ipfw add pipe 1 ip from any to 192.168.0.101 out recv em0
      ipfw add skipto 10000 ip from any to 192.168.0.101 out recv em0
      # Upstream
      ipfw pipe 2 config bw 896kbits/s
      ipfw add pipe 2 ip from 192.168.0.101 to any out recv em1
      ipfw add skipto 10000 ip from 192.168.0.101 to any out recv em1

      # Host Group #2
      # Downstream
      ipfw pipe 3 config bw 3mbits/s
      ipfw add pipe 3 ip from any to $host_group_1 out recv em0
      ipfw add skipto 10000 ip from any to $host_group_1 out recv em0
      # Upstream
      ipfw pipe 4 config bw 500kbits/s
      ipfw add pipe 4 ip from $host_group_1 to any out recv em1
      ipfw add skipto 10000 ip from $host_group_1 to any out recv em1

      # Limit all other hosts with different limits
      # Downstream
      ipfw pipe 5 config bw 25mbits/s
      ipfw add pipe 5 ip from any to any out recv em0
      # Upstream
      ipfw pipe 6 config bw 10mbits/s
      ipfw add pipe 6 ip from any to any out recv em1

      # Needed to pass traffic after being processes by dummynet/pipe.
      ipfw add 10000 allow ip from any to any

  If you get unexpected behavior for this above configuration, such as rules not processing, ensure the `sysctl net.inet.ip.fw.one_pass` is set to "0".


Troubleshooting
----
- You will undoubtedly receive errors about the secondary GPT table being corrupt for the installation media. This is inherent when writing an image to a non-exact sized thumbdrive. This error can safely be ignored but if you want to repair the secondary GPT table run the following command:

      # gpart recover /dev/da1

- If during boot of the installation media the script drops to a `mountroot>`, modify the `/boot/loader.conf` file to include `kern.cam.boot_delay="10000"` to cause a 10 second delay while booting. See below for context.

      uhub2: 4 ports with 4 removable, self powered
      mountroot: waiting for device /dev/ufs/FreeBSD_Install...
      Mounting from ufs:/dev/ufs/FreeBSD_Install failed with error 19.

      Loader variables:
      vfs.root.mountfrom=ufs:/dev/ufs/FreeBSD_Install
      vfs.root.mountfrom.options=ro,noatime

      Manual root filesystem specification:
        <fstype>:<device> [options]
            Mount <device> using filesystem <fstype>
            and with the specified (optional) option list.

          eg. ufs:/dev/da0s1a
              zfs:tank
              cd9660:/dev/cd0 ro
                (which is equivalent to: mount -t cd9660 -o ro /dev/cd0 /)

        ?               List valid disk boot devices
        .               Yield 1 second (for background tasks)
        <empty line>    Abort manual input

      mountroot>

Email me with other issues to post here.


Thank you
----
This project was made possible by Luigi Rizzo for creating the code and Allan Jude for making me aware of the technology on [BSD Now](http://bsdnow.tv).


References
----
- [FreeBSD Handbook - 29.4. IPFW](https://www.freebsd.org/doc/handbook/firewalls-ipfw.html)
- [FreeBSD Manual Pages - ipfw(8)](https://www.freebsd.org/cgi/man.cgi?ipfw(8))
- [FreeBSD Handbook - 30.6. Bridging](https://www.freebsd.org/doc/handbook/network-bridging.html)
- [CS.ECS.Baylor.edu - Michael J. Donahoo - Basic DummyNet Tutorial](http://cs.ecs.baylor.edu/~donahoo/tools/dummy/tutorial.htm)
- [Luigi Rizzo -- research](http://info.iet.unipi.it/~luigi/research.html)
- [Luigi Rizzo -- The dummynet project page](http://info.iet.unipi.it/~luigi/dummynet/)
