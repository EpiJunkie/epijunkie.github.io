---
layout: post
title: Running poudriere in bhyve and on bare metal
date: 2016-07-03
tags: bhyve poudriere
category: blog
---

# Running the same poudriere instance in bhyve and on bare metal

This article goes over running [poudriere](https://github.com/freebsd/poudriere) to built packages for a Raspberry Pi with the interesting twist of running it both as a bhyve guest and then switching to running on bare metal via Fiber Channel via [ctld](https://www.freebsd.org/cgi/man.cgi?query=ctld&sektion=8) by sharing the same ZFS volume. A `nginx` server is also configured to give a status report of the `poudriere` build process.

This guide assume you have an existing host running FreeBSD 10.3+, all other configuration points are covered in this article including installing FreeBSD on a Raspberry Pi. This also assumes you have two Qlogic Fiber Channel cards, see [the bottom of this linked page for suggested cards](http://justinholcomb.me/blog/2014/03/12/fiberchannel.html).

## Why would a person want to do this?

Firstly, `poudriere` can build packages for different architectures such as ARM. This can save hours of build time compared to building ports from said ARM device.

Secondly, let's say a person has an always-on device (NAS) running FreeBSD. To save power, this device has a CPU with a low clock-rate and low core count. This low clock-rate and core count is great for saving power but terrible for processor intensive application such as poudriere. Let's say a person also has another physical server with fast processors and a high CPU count but draws nearly twice the power and a fan noise to match.

To get the best of both worlds, the goal is to build the packages on the fast physical server, power it down, and then start the same ZFS volume in a bhyve environment to serve packages from the always-on device.

## Overview

Hostnames:
* "`ahost`" - Always-on physical host
* "`fhost`" - Fast, noisy, powerful physical host
* "`geodude`" - Poudriere guest
* "`rpi`" - Raspberry Pi

Simplified process:
* Prepare `ahost`
* Prepare `geodude` in bhyve and shutdown
* Build packages on `fhost` and shutdown
* Start `geodude` in bhyve to serve packages
* Install image on raspberry pi, configure, and point to poudriere repo.
* Do something with the spare time you now have

## Details

### Preparation of `ahost`

#### Create a bhyve guest disk

`zfs create -V 64G dozer/bhyve/geodude/disk0`

#### Configure `ctld`
The `ctld` daemon is used to present the above ZFS volume over Fiber Channel on the first port (`isp0`) on LUN slot 12. This is done by creating the `/etc/ctld.conf` file and adding the following:
```
target naa.210200e08bdb1d3e {
  alias fhost
  auth-group no-authentication
  port isp0

  lun 12 {
    backend block
    path /dev/zvol/dozer/bhyve/geodude/disk0
  }
}
```

To get the network address authority (naa) name run `ctladm port -l` after both ends of the fiber are illuminated.

It will look something like this:
```
root@themoat:~ # ctladm port -l
Port Online Frontend Name     pp vp
0    YES    tpc      tpc      0  0  
1    NO     camsim   camsim   0  0  naa.5000000931975702
2    YES    ioctl    ioctl    0  0  
3    YES    camtgt   isp0     0  0  naa.210200e08bdb1d3e
4    NO     camtgt   isp1     0  0  
5    NO     camtgt   isp2     0  0  
6    NO     camtgt   isp3     0  0  
```

#### Build custom kernel
This is necessary because switching the Fiber Channel mode to "target" is impossible otherwise.

Create a new kernel configuration file based off of GENERIC called FCTARGET
```
cd /usr/src/sys/amd64
cp GENERIC FCTARGET
```

Add and remove lines as follows in the `/usr/src/sys/amd64/FCTARGET` kernel configuration:
```
- ident           GENERIC
+ ident           FCTARGET
- #device         ispfw                   # Firmware for QLogic HBAs
+ device          ispfw                   # Firmware for QLogic HBAs
+ options         ISP_TARGET_MODE         # Needed for Target Mode for Fiber Channel
```

Build, install, and boot from the new kernel:
```
make buildkernel KERNCONF=FCTARGET
make installkernel KERNCONF=FCTARGET
reboot
```

### Preparation of `geodude`

#### Install FreeBSD on `geodude` in bhyve

Start `geodude` using the slimmed down boot only FreeBSD 10.3 installation ISO:
```
bhyveload -m 2G -d /chyves/ISO/FreeBSD-10.3-RELEASE-amd64-bootonly.iso/FreeBSD-10.3-RELEASE-amd64-bootonly.iso -c /dev/nmdm0A geodude
bhyve -A -H -P -c 2 -m 2G -s 0,hostbridge -s 3,ahci-cd,/chyves/ISO/FreeBSD-10.3-RELEASE-amd64-bootonly.iso/FreeBSD-10.3-RELEASE-amd64-bootonly.iso -s 4,ahci-hd,/dev/zvol/dozer/bhyve/geodude/disk0 -s 5,virtio-net,tap1 -l com1,/dev/nmdm0A -s 31,lpc geodude
```

Install a ZFS based layout with swap turned on and include the 32-bit library and source packages.

Start `geodude` after the installation:
```
bhyveload -m 2G -d /dev/zvol/dozer/bhyve/geodude/disk0 -c /dev/nmdm0A geodude
bhyve -A -H -P -c 2 -m 2G -s 0,hostbridge -s 4,ahci-hd,/dev/zvol/dozer/bhyve/geodude/disk0 -s 5,virtio-net,tap1 -l com1,/dev/nmdm0A -s 31,lpc geodude
```

#### Install packages on `geodude`:

Run `portsnap fetch extract` to update to the latest ports. Then install the following ports:

poudriere-devel:
```
cd /usr/ports/ports-mgmt/poudriere-devel/
make config-recursive                                   # Enable the qemu-arm-static option for poudriere-devel. All else can use defaults.
make install clean
```

nginx
```
cd /usr/ports/www/nginx/
make config-recursive                                   # Defaults are fine
make install clean
```

#### Configure `poudriere`

Configure `poudriere` with the minimum options in `/usr/local/etc/poudriere.conf`:
```
ZPOOL=zroot
FREEBSD_HOST=http://ftp.FreeBSD.org/pub/FreeBSD/
NOLINUX=yes
```

Turn on the ability to emulate an ARMv6 architecture through QEMU (this is not a persistent and does not survive between reboots):
```
binmiscctl add armv6 --interpreter "/usr/local/bin/qemu-arm-static" --magic "\x7f\x45\x4c\x46\x01\x01\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x02\x00\x28\x00" --mask "\xff\xff\xff\xff\xff\xff\xff\x00\xff\xff\xff\xff\xff\xff\xff\xff\xfe\xff\xff\xff" --size 20 --set-enabled
```

Create a 10.3-RELEASE jail for poudriere to build the ARMv6 packages:
```
poudriere jail -c -j 103armv6 -m svn -a arm.armv6 -v release/10.3.0
```

Seed default ports tree:
```
poudriere ports -c
```

Create a text file (`pkglist.txt`) with the desired ports at `/usr/local/etc/poudriere.d/`. One package per line in `category/port` format:
```
sysutils/tmux
editors/nano
www/nginx
security/py-fail2ban
www/rubygem-jekyll
```

Select the options for each port (this is like running `make config-recursive` but for all the ports and dependancies in the pkglist.txt):
```
poudriere options -cf /usr/local/etc/poudriere.d/pkglist.txt
```

#### Configure nginx

Enable `nginx` to start on boot
```
sysrc nginx_enable=yes
```

Create folder and file for `nginx` logs:
```
mkdir /usr/local/etc/nginx/logs
touch /usr/local/etc/nginx/logs/access.log
```

Configure nginx via the `/usr/local/etc/nginx/nginx.conf` file:
```
worker_processes  1;

events {
    worker_connections  1024;
}

http {
  include       mime.types;
  default_type  application/octet-stream;

  log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

  access_log  logs/access.log  main;

  sendfile        on;

  keepalive_timeout  65;

  server {
    listen       80;
    server_name  geodude, fhost;
    root         /usr/local/share/poudriere/html;

    # Allow caching static resources
    location ~* ^.+\.(jpg|jpeg|gif|png|ico|svg|woff|css|js|html)$ {
      add_header Cache-Control "public";
      expires 2d;
    }

    location /data {
      alias /usr/local/poudriere/data/logs/bulk;

      # Allow caching dynamic files but ensure they get rechecked
      location ~* ^.+\.(log|txz|tbz|bz2|gz)$ {
        add_header Cache-Control "public, must-revalidate, proxy-revalidate";
      }

      # Don't log json requests as they come in frequently and ensure
      # caching works as expected
      location ~* ^.+\.(json)$ {
        add_header Cache-Control "public, must-revalidate, proxy-revalidate";
        access_log off;
        log_not_found off;
      }

      # Allow indexing only in log dirs
      location ~ /data/?.*/(logs|latest-per-pkg)/ {
        autoindex on;
      }

      break;
    }

    location /repo {
      alias /usr/local/poudriere/data/packages;
    }
  }
}
```

Check the configuration and start:
```
service nginx configtest
service nginx start
```

Navigate in a web-browser to http://geodude/ or the IP to verify an empty `poudriere` web interface is running.

You may also want to set the configuration of the IP address in `/etc/rc.conf` for the physical interfaces on `fhost`. You can have both net interfaces configured in `rc.conf` _without_ needing to comment out the interfaces for the method you are booting `geodude`.

Power down `geodude` once this is done.

### Run `geodude` on `fhost`

Start the `ctld` service on `ahost` to boot `geodude` on `fhost`.
```
service ctld onestart
```

Start `fhost` and enter the Qlogic BIOS configuration utility by pressing "Ctrl + q" when displayed during boot. Then go through the menu system as follows:
* Configuration Settings
  * Selectable Boot Settings
    * Selectable Boot: enabled
    * (Primary) Boot Port Name,Lun: &#60;ENTER&#62;
      * 0 FREEBSD CTLDISK 0001 &#60;ENTER&#62;
        * (ONLY IF MORE THAN ONE LUN IS CONFIGURED) Select LUN 12 to be the boot device.
      * &#60;ESC&#62;
    * &#60;ESC&#62;
  * Save changes
* Exit Fast!UTIL

`fhost` should now start booting from the `geodude` disk, the boot order may need to be changed or manually selected.

#### Build packages
Turn on ARMv6 architecture emulation again through QEMU:
```
binmiscctl add armv6 --interpreter "/usr/local/bin/qemu-arm-static" --magic "\x7f\x45\x4c\x46\x01\x01\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x02\x00\x28\x00" --mask "\xff\xff\xff\xff\xff\xff\xff\x00\xff\xff\xff\xff\xff\xff\xff\xff\xfe\xff\xff\xff" --size 20 --set-enabled
```

Start the build process of the desired packages:
```
poudriere bulk -j 103armv6 -f /usr/local/etc/poudriere.d/pkglist.txt
```

...wait until build process completes. Status of the build process can be checked via the nginx web interface at http://fhost/.

#### Stop `fhost`

Shutdown `fhost` when build process is complete.
```
poweroff
```

### Serve packages from `geodude` while running in `bhyve` on `ahost`

Shutdown `ctld` service on `ahost` (ZFS Volume will be locked to `ctld` until this is turned off.):
```
service ctld onestop
```

Start `geodude` in a `bhyve` environment again:
```
bhyveload -m 2G -d /dev/zvol/dozer/bhyve/geodude/disk0 -c /dev/nmdm0A geodude
bhyve -A -H -P -c 2 -m 2G -s 0,hostbridge -s 4,ahci-hd,/dev/zvol/dozer/bhyve/geodude/disk0 -s 5,virtio-net,tap1 -l com1,/dev/nmdm0A -s 31,lpc geodude
```

### Raspberry Pi

#### Install Raspberry Pi image to SD card:

[Download the latest Raspberry Pi image from the SD Card Images section](https://www.freebsd.org/where.html), decompress it and pipe to `dd` to the micro-SD card. This should go without saying but `da1` should be your micro-SD card for the Raspberry Pi.
```
fetch ftp://ftp.freebsd.org/pub/FreeBSD/releases/arm/armv6/ISO-IMAGES/10.3/FreeBSD-10.3-RELEASE-arm-armv6-RPI-B.img.xz
xzcat FreeBSD-10.3-RELEASE-arm-armv6-RPI-B.img.xz | dd of=/dev/da1 bs=1M
```

#### Configure Raspberry Pi to point to poudriere repo

Edit `/etc/pkg/FreeBSD.conf` as follows:
```
FreeBSD: {
   url: "pkg+http://geodude/repo/103armv6-default",
   mirror_type: "srv",
   signature_type: "none",
   enabled: true
}
```

#### Update packages
```
pkg update
```

#### Install packages
```
pkg install tmux nano nginx py27-fail2ban rubygem-jekyll
```

## Final thoughts
This is largely a proof-of-concept but does demonstrate an easy way to save time and electricity when building packages with `poudriere` for ARM based devices.

## Appendix

### Website sources
Doug Vetter's [Building ARMV6 Ports on FreeBSD with Poudriere](https://www.dvatp.com/tech/armv6_freebsd_poudriere)

### Hardware
* Raspberry Pi 2 B+
* Dell R610 (`ahost`)
  * 2x Intel Xeon [L5630](http://ark.intel.com/products/47927/) processors
    * Each with 4 Cores with HT @ 2.13Ghz
  * Qlogic QLE2464
    * Directly connected to `fhost` via OM3 LC–LC Fiber
* Dell R610 (`fhost`)
  * 2x Intel Xeon [X5650](http://ark.intel.com/products/47922/) processors
    * Each with 6 Cores with HT @ 2.66Ghz
  * Qlogic QLE2460
    * Directly connected to `ahost` via OM3 LC–LC Fiber
