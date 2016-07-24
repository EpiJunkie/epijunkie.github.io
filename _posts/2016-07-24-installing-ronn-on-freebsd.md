---
layout: post
title: Installing ronn on FreeBSD
date: 2016-07-24
tags: chyves ronn man page
category: blog
---

These are some quick notes on getting `ronn` and it's dependancies installed on FreeBSD. Next entry here, in a couple of weeks, will have some interesting PCI pass-through related fun.

# Installing Ruby tool '`ronn`' on FreeBSD
This weekend I converted `chyves` to use the Ruby tool [ronn](https://github.com/rtomayko/ronn) as the man page generator for the project. Previously `txt2man` was being used but it lacked the ability to control the formatting and also the complexity of the `chyves` man page exceeded `txt2man`'s intended simple use case. There are now 2,200 lines in the `chyves` man page from 487 at the time of forking.

Install `bash`:

```
pkg install bash
```

Add the file-descriptor file system entry to `/etc/fstab`:

```
fdesc    /dev/fd     fdescfs     rw  0   0
```

Mount the new entry:

```
mount -a
```

Enter into the `bash` shell:

```
bash
```

Download the Ruby Version Manager:

```
fetch https://get.rvm.io -o ./installer.sh
```

Review the shell script you just downloaded from the interwebs and make sure you are willing to execute that code as root against your most precious data. Seriously, I'll wait...

Run that shell script you just reviewed:

```
bash installer.sh stable
```

Import the RVM environment:

```
source /etc/profile.d/rvm.sh
```

Install Ruby version 2.2.5

```
rvm install 2.2.5
```

Install the Ruby gem `ronn`:

```
gem install ronn
```

Now you can use the `make docs` directive from the `chyves` source directory after making changes to rebuild the documentation files... because laziness.

# Sources
[Digital Ocean - How To Install Ruby on Rails on FreeBSD 10.1 using RVM](https://www.digitalocean.com/community/tutorials/how-to-install-ruby-on-rails-on-freebsd-10-1-using-rvm)

[AtomicObject.com - Creating Man Pages in Markdown with Ronn](https://spin.atomicobject.com/2015/05/06/man-pages-in-markdown-ronn/)

[RicoStaCruz.com - The great cheatsheet for ronn](http://ricostacruz.com/cheatsheets/ronn.html)
