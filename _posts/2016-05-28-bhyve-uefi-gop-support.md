---
layout: post
title: bhyve now supports UEFI GOP
date: 2016-05-28
tags: freebsd bhyve
category: blog
---

# bhyve now supports UEFI GOP aka VNC support for guests

Initial graphics support (UEFI GOP) has been released for `bhyve` for UEFI  guests. Rather than interfacing with a guest through a serial interface, a basic VNC server is started to manage the guest. For Windows this negates the need to use the [AutoAttend.xml installation method](https://people.freebsd.org/~grehan/bhyve_uefi/windows_iso_repack.txt)  and the boot process can be watched which is helpful during update installations. Matt Churchyard has already added support to [vm-bhyve](https://github.com/churchers/vm-bhyve) for these recent developments. After taking a look at the [source changes to bhyve](https://svnweb.freebsd.org/base?view=revision&revision=300829) and looking at the work from Matt Churchyard, I have been able to figure out how to implement this directly in bhyve.

The VNC server is attached to a frame buffer as a PCI device. When `bhyve` is started with a frame buffer device, a PS2 mouse and keyboard are automatically attached. For newer OSes, also attaching a xHCI (USB 3.0) tablet is suggested as the PS2 mouse does not map the position of the cursor exactly. The screen resolution, the binding IP, and listening port can be configured, see examples below. There is also the `wait` directive which waits to boot the guest until a VNC client connects, helpful for installations.

## Beware

There are some rough edges with this. This is _not_ being developed by a multi-national-billion-dollar company nor has it been in development for the last decade, so keep that in mind before demanding features. At the very least reach out to the developers and thank them.

One of the rough edges: the VNC server seems to be temperamental to which client connects to it. On OSX I had success with [Chicken of the VNC](https://sourceforge.net/projects/chicken/). Interestingly (on OSX) I was able to install an OS on a guest using the [RealVNC](https://www.realvnc.com/) client but after the installation it would no longer connect. I would venture to guess it is due to the compression method used by the client/server and negotiating which compression encoding to use.

## Requirements
- Updated [`bhyve` binary with graphics support](http://svn.freebsd.org/base/projects/bhyve_graphics).
- New UEFI firmware from Peter Grehan - [BHYVE_UEFI_20160526.fd](https://people.freebsd.org/~grehan/bhyve_uefi/BHYVE_UEFI_20160526.fd)
- FreeBSD 11-CURRENT (Tested against r300097
[ISO](http://ftp.freebsd.org/pub/FreeBSD/snapshots/ISO-IMAGES/11.0/http://ftp.freebsd.org/pub/FreeBSD/snapshots/ISO-IMAGES/11.0/))
- UEFI guest

## How to test
This feature requires 11-CURRENT and was tested against a refresh installation of the [r300097 ISO snapshot](http://ftp.freebsd.org/pub/FreeBSD/snapshots/ISO-IMAGES/11.0/FreeBSD-11.0-CURRENT-amd64-20160518-r300097-disc1.iso.xz). An [upgraded system worked as well](http://www.bsdnow.tv/tutorials/stable-current) and was updated to r300899.

After booting the 11-CURRENT host, download the bhyve-graphics code, build  `bhyve`, replace the current `bhyve` binary in `$PATH`, and then download the latest UEFI boot binary from grehan's public FreeBSD file directory.

```
svnlite co http://svn.freebsd.org/base/projects/bhyve_graphics
cd bhyve_graphics
make BHYVE_SYSDIR=/usr/src -m /usr/src/share/mk
cp bhyve /usr/sbin/bhyve
fetch https://people.freebsd.org/~grehan/bhyve_uefi/BHYVE_UEFI_20160526.fd
```

## Examples
Below are examples, each with the frame buffer being attached to slot 5:

### Base:

```
bhyve \
      -c 2 \
      -s 0,hostbridge \
      -s 3,ahci-hd,/images/win.img \
      -s 4,ahci-cd,/images/win_repack.iso \
      -s 5,fbuf \
      -s 10,virtio-net,tap0 \
      -s 31,lpc \
      -l bootrom,/path/to/BHYVE_UEFI_20160526.fd \
      -m 2G -H -w \
      windows-guest
````


### Wait:
Requires a VNC client connect before booting. Helpful when a key press is necessary to start an installation disc.

```
bhyve \
      -c 2 \
      -s 0,hostbridge \
      -s 3,ahci-hd,/images/win.img \
      -s 4,ahci-cd,/images/win_repack.iso \
      -s 5,fbuf,wait \
      -s 10,virtio-net,tap0 \
      -s 31,lpc \
      -l bootrom,/path/to/BHYVE_UEFI_20160526.fd \
      -m 2G -H -w \
      windows-guest
````


### Resolution:
Used to specify the resolution (1920x1080) when desiring something else than the default 800x600 resolution.

```
bhyve \
      -c 2 \
      -s 0,hostbridge \
      -s 3,ahci-hd,/images/win.img \
      -s 4,ahci-cd,/images/win_repack.iso \
      -s 5,fbuf,w=1920,h=1080 \
      -s 10,virtio-net,tap0 \
      -s 31,lpc \
      -l bootrom,/path/to/BHYVE_UEFI_20160526.fd \
      -m 2G -H -w \
      windows-guest
````

#### Supported resolutions
- 1920 x 1200
- 1920 x 1080
- 1600 x 1200
- 1600 x 900
- 1280 x 1024
- 1280 x 720
- 1024 x 768
- 800 x 600 (Default)
- 640 x 480


### Port:
Used to specify the port (5900) to listen on. When not specified, a random port is selected in which case you will need to run `sockstat | grep bhyve` to figure out which port was selected.

```
bhyve \
      -c 2 \
      -s 0,hostbridge \
      -s 3,ahci-hd,/images/win.img \
      -s 4,ahci-cd,/images/win_repack.iso \
      -s 5,fbuf,tcp=0.0.0.0:5900 \
      -s 10,virtio-net,tap0 \
      -s 31,lpc \
      -l bootrom,/path/to/BHYVE_UEFI_20160526.fd \
      -m 2G -H -w \
      windows-guest
````


### IP:
Used to specify the IP (192.168.1.100) to bind to. When not specified, VNC server is bound to all IPs (0.0.0.0).

```
bhyve \
      -c 2 \
      -s 0,hostbridge \
      -s 3,ahci-hd,/images/win.img \
      -s 4,ahci-cd,/images/win_repack.iso \
      -s 5,fbuf,tcp=192.168.1.100 \
      -s 10,virtio-net,tap0 \
      -s 31,lpc \
      -l bootrom,/path/to/BHYVE_UEFI_20160526.fd \
      -m 2G -H -w \
      windows-guest
````


### xHCI mouse:
Used to attach a USB 3.0 tablet for better mouse tracking (Only compatible for newer OSes supporting USB 3.0). In this example, it is attached on PCI slot 6.

```
bhyve \
      -c 2 \
      -s 0,hostbridge \
      -s 3,ahci-hd,/images/win.img \
      -s 4,ahci-cd,/images/win_repack.iso \
      -s 5,fbuf \
      -s 6,xhci,tablet \
      -s 10,virtio-net,tap0 \
      -s 31,lpc \
      -l bootrom,/path/to/BHYVE_UEFI_20160526.fd \
      -m 2G -H -w \
      windows-guest
````


### Using all options:
Example of when all the directives are used.

```
bhyve \
      -c 2 \
      -s 0,hostbridge \
      -s 3,ahci-hd,/images/win.img \
      -s 4,ahci-cd,/images/win_repack.iso \
      -s 5,fbuf,tcp:192.168.1.100:5900,w=1920,h=1080,wait \
      -s 6,xhci,tablet \
      -s 10,virtio-net,tap0 \
      -s 31,lpc \
      -l bootrom,/path/to/BHYVE_UEFI_20160526.fd \
      -m 2G -H -w \
      windows-guest
```

## Thanks
- Peter Grehan
- Leon Dang
- Tycho Nightingale
- Michael Dexter
- Matt Churchyard

...for developing and testing this sweet `bhyve` feature.

## Resources
- [Matt Churchyard's vm-bhyve - UEFI Graphics Page](https://github.com/churchers/vm-bhyve/wiki/UEFI-Graphics-%28VNC%29)
- [bhyve graphics commit](https://svnweb.freebsd.org/base?view=revision&revision=300829)
- [Base bhyve UEFI command from here](https://people.freebsd.org/~grehan/bhyve_uefi/windows_install.txt)
- [BSDNow.tv - Tutorial - Tracking -STABLE and -CURRENT (FreeBSD)](http://www.bsdnow.tv/tutorials/stable-current)

## Testing platform
- Dell Poweredge R610, dual L5630 Xeon
- Fresh install of -CURRENT r300097 from ISO snapshots
  - Then ran further test after following the BSDNow.tv guide to upgrade to -CURRENT r300899
    - Took about 80 minutes to `buildworld` against 16 logical cores at 2.13Ghz.
- Successfully tested the following OSes:
  - Windows 2012r2 Standard Server from MSDN image
  - Windows 8.1 from MSDN image
  - Windows 10 from MSDN image
    - Booted to the installation disc but Windows did not like the environment and the installation would not boot.
  - FreeBSD 10.3-REL UEFI DVD
