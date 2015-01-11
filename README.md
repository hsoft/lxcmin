# lxcmin - Create minimal LXC containers with Gentoo

I'm toying with LXC and Gentoo, trying to find a convenient way to easily generate minimal LXC
containers. You know, containers without a whole distribution with it?

My benchmark is Docker's `ubuntu:trusty` image which, when we `du -hs` it, goes up to `265M`.

So far, what I've accomplished is that I managed generate a LXC image that runs bash, from scratch,
using Portage.

For now, I only have textual instructions, but with luck, this will evolve in a full-blown app.

## Instructions

To follow these instructions, you need to be running Gentoo and have `app-emulation/lxc` installed.

### 1 - Be root

I'm more used to Docker than LXC. I saw that it's possible to run LXC as a non-root, but I haven't
tried that yet.

### 2 - Create a sandbox directory

    $ mkdir /lxctest
    $ cd /lxctest

Then, let's create our `rootfs`, which will be the root directory of our container.

    $ mkdir rootfs

### 3 - The config file

I haven't played too much with that, but here's what I put in `/lxctest/config`:

    lxc.network.type = empty
    lxc.rootfs = /lxctest/rootfs
    lxc.utsname = foo
    lxc.autodev = 1
    lxc.tty = 1
    lxc.pts = 1

### 4 - Create a portage config

Unless your host Gentoo config is minimal, you'll need to create an alternative config to build
your container. For example, if you're set up with `systemd`, you'll have problems creating a
minimal image because there's a *lot* of stuff that comes with the systemd profile.

What I did is that I created a `/lxctest/portage-config/etc/portage` folder and I copied the
contents of my `/etc/portage` folder in it. Then, I changed my `make.profile` symlink to
`/usr/portage/profiles/default/linux/amd64/13.0/no-multilib` and cleaned up my `make.conf` to
remove as much stuff as possible.

Tweak at will.

### 5 - Emerge!

We're now at the fun part, we emerge a base system in our sandbox. This is done with the use of
the `ROOT` and `PORTAGE_CONFIGROOT` environment variables. First, let's start with our base layout:

    $ ROOT=/lxctest/rootfs PORTAGE_CONFIGROOT=/lxctest/portage-config emerge baselayout

This creates our base directory hierarchy and should be emerged first. In my experimentations,
when it wasn't emerged first, then it was pulled as a dependency at some point and its installation
failed because some of the directories it creates were already there.

Then, we can go with the real stuff:

    $ ROOT=/lxctest/rootfs PORTAGE_CONFIGROOT=/lxctest/portage-config emerge bash lxc

Why `lxc` because our container needs `lxc-init` for `lxc-execute` to work well. I'm not sure how
it all works yet. If you only plan to use `lxc-start`, I don't think that the `lxc` package is
needed. I'm not sure yet.

In my tests, my first `lxc-execute didn't work because I didn't have libseccomp. I installed it
manually:

    $ ROOT=/lxctest/rootfs PORTAGE_CONFIGROOT=/lxctest/portage-config emerge sys-libs/libseccomp

### 6 - Last minutes tweaks

If you try to run your container now, you'll have errors about `/dev`, `/proc` and `/sys` not
existing. You simply have to `mkdir` them:

    $ mkdir rootfs/dev
    $ mkdir rootfs/proc
    $ mkdir rootfs/sys

### 7 - Run!

You're ready to go!

    $ lxc-execute -n lxcmin -f /lxctest/config /bin/bash

Along with a few warning messages (I haven't figured them out yet), you will have a bash prompt
inside your container! How minimal is it?

    # ls
    bash: ls: command not found

How big is it?

    $ du -hs /lxctest/rootfs
    99M /lxctest/rootfs

