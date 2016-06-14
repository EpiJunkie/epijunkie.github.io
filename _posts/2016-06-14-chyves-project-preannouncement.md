---
layout: post
title: chyves pre-announcement
date: 2016-06-14
tags: chyves bhyve iohyve
category: blog
---

# chyves pre-announcement
This is a pre-announcement for the chyves project.

`chyves` manages `bhyve` guests using ZFS, `nmdm`, `virtio`, `cu` and optionally `netmap`, `tmux` and `grub2-bhyve` through the Bourne shell. chyves is a bhyve front end manager. chyves follows the lead set by iocage by storing guest parameters as ZFS user properties. `chyves` is a direct code descendent of `iohyve` with history maintained from where it was split at version 0.7.5, commit [2ff5b50](https://github.com/pr1ntf/iohyve/tree/2ff5b50d8cda61a8364bd79319152142ac1b4c33). A nearly complete code rewrite was started in April 2016 and as of this post, the rewrite is on-going with code being written/committed nearly daily.

## History

Earlier this year I started using `bhyve`, actually I started using `iohyve`. I wanted to starting using `bhyve` but did not fully understand how it all worked. I did not know what all the knobs, flags, and how they interacted with the tools it depended on. I am a big-picture type of person and sometimes that means working backwards to figure out how something works. I needed to see it working and then I needed to work backwards to understand every detail. So I started using `iohyve` because it seemed to be the simplest of the `bhyve` managers. There also seemed to be some automation happening behind the scenes. By that I mean, there was far less configuration needed, many of the other `bhyve` front-end managers seemed to parse an input file to `bhyve`. I also started using `iohyve` because something about storing guest properties in ZFS user properties is incredibly appealing to me. [James May](https://en.wikipedia.org/wiki/James_May) once described a [fizzing feeling](http://www.topgear.com/car-news/james-may/james%E2%80%99s-fizzy-logic) on national UK television, I would say utilizing ZFS user properties is nearly the same for me.

However one day I entered the command "`iohyve delete pfSense`" to delete my retired gold base image that was getting replaced with a newer version. But then all my VMs starting with "pfSense" were deleted. I was livid, mostly at myself for not backing up the config on pfSense from the day before. I lost a few hours of work and had that looming sense of dread from potentially forgetting something in the recreated firewall config. I checked out the `iohyve` code and saw the problem. The [delete function was loosely written](https://github.com/pr1ntf/iohyve/blob/1754129465ea144948b60b5f53dc0241793ac412/iohyve#L821) and guests sharing the same base name would all be deleted. I suspect this was done to delete the child datasets, my guess is the `-r` flag used with `zfs destroy` was not fully understood.

This incident generated enough angst that I submitted [my first ever Pull Request (PR)](https://github.com/pr1ntf/iohyve/pull/119). That was my first public submission of code of any kind, I felt a little proud actually; a historic moment in that personal-meatspace-milestone-achievement type of way. Over the next couple of weeks I intentionally tried to get `iohyve` to break and reveal bugs. I found quiet a few issues and [submitted PRs](https://github.com/pr1ntf/iohyve/issues?utf8=%E2%9C%93&q=author%3Aepijunkie) to address them. I decided to fork my own project because the rate my code was being reviewed and ingested was too slow for the number of changes I wanted to implement. I also wanted to make some huge changes and had already met resistance for smaller changes. At any rate, Trent pointed out the project was release under the BSD Clause-2 License so I did not need his permission but he gave his "okay" anyways.

I am incredibly grateful to Trent for creating `iohyve` and to everyone who has contributed to `iohyve` as well.

## What is to come:
Here are just a few things coming with `chyves`. Keep in mind, _this vague list is code that has **already** been written/committed_. There is more to come but until I write it, I am not announcing it.

### End user features:
- Global configuration
  - New guest properties are no longer hard coded and can be changed.
  - There are now properties for the way `chyves` runs
  - Properties for network configuration
    - Which `tap` belong to which `bridge`s
    - Which (if any) physical/vlan interface belong to which `bridge`s.
- Added new guest properties and deprecated others.
- Multi-pool support is an intentional part of `chyves`
- Multiple guests can be specified in certain commands.
- Consolidated many sub-commands into having sub-sub-commands. Make it far easier to remember.
  - I can't tell you how many times I thought _"Is it `chyves listfw` or `chyves fwlist`"_, it is now `chyves firmware list`.
- Consolidated the `start`, `install`, `uefi` commands into `start`.
- Guest reboots are handled correctly and consistent to how a reasonable person would expect.
- Dynamic generation of bhyve PCI strings for all device types.
  - Addtional hard disk no longer need to be hard coded with a PCI device.
- Complete rewrite of ISO and Firmware handling.
  - Hash checking and decompression of images for ISOs
- Rewrote the way networking is handled
  - `bridge0` is no longer hard coded
  - VALE support, not entirely stable but hopefully this will give more exposure and attrack more developers for VALE/netmap support in bhyve.
  - Complex networks can be designed with multiple taps, multiple bridges, multiple VALE switches, and multiple vlan/physical interfaces.
  - Simple networks only need to have a physical/vlan interface associated with bridge0 or whatever you choose to be the default bridge. `bridge0` is still the initial-default.
  - `chyves setup net=em0` has been replaced by `chyves network bridge{n} attach em0`
    - This tells `chyves` to attach the physical interface em0 to bridge{n}.
  - Private bridges are a configuration option for sensitive traffic, such as backend SQL databases.
  - Network design is created and verified from `chyves` properties when starting a guest.
  - A manual mode is available.
- Extensively expanded man page
- Wrote checks for:
  - Number of parameter fields are met and not exceeded
  - CPU features checked
  - IO MMU check when starting guest with pass-through devices.
  - Input verification
    - Network sub-command classifies user input for interfaces into valid types.
    - New parsing structure
	- Properties can only be set to valid values
- VNC support for UEFI guests.

### Internal changes:
- Nearly every function from `iohyve` has been rewritten, replaced, or deprecated.
- 50+ additional functions have been created. Many of these are to validate user input, automate processes, or generally improve the user and developer experience.

### Developer targeted changes:
- Dev mode can be turned on:
  - `dev` sub-command to execute functions directly and optionally execute it with the Bourne `-x` `-v` and or `-n` flags
  - The exact `bhyve` commands used to start guests are displayed while starting the guest.
- More `Makefile` actions
- Standardized whitespace

### Project changes:
- Structured to be a community effort, instead of any single person's project.
- More project documentation
- But most of all we have a cool logo now:

<center><img src="https://raw.githubusercontent.com/chyves/chyves-media/master/chyves-logo-v1-xlarge.png" alt="chyves logo version 1" width="449" height="439" align="center"></center>

## Community effort
So I just mentioned community effort but then am keeping the repo private until the initial release. That is a total contradiction I admit, my thought is I want a solid base before opening up the code. I have also been making frequent changes to the Global properties and do not want to be tracking another point with all these massive changes to the code.

## When?!?!
Intended release time frame is mid-August 2016.

## Demonstration
The project website has demonstrations at [http://chyves.org](http://chyves.org/#demo).
