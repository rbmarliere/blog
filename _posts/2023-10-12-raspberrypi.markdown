---
layout: post
title: "Booting a minimal upstream kernel in a Raspberry Pi 3"
date: 2023-10-12 10:05:00 -0300
image: /~ricardo/assets/rpi.png
tags: linux kernel mentorship
---

In this post, I will go through the steps for building and booting a [linux-next][next]
kernel on that board you have gathering dust somewhere.

![raspberry](/~ricardo/assets/rpi.png)

--- **DISCLAIMER**: *The contents of this post may be severely outdated by the time you read
them.*

Back in my [Funtoo][funtoo] [^1] days, I wrote a nasty [script][sysroot] to organize the
steps required to have a functioning cross-compilation toolchain in order to build the
[vendor][rpi-kernel] kernel. It had all sort of weird things that I wanted to learn,
such as embedding a dropbear ssh server in the initramfs and filesystem encryption. This
time, I wanted to try a different approach: what is the minimum possible setup I can
come up with to properly boot a root filesystem on the Pi?

If you followed along my [last post][better-workflow], you noticed that I chose to
configure a crash kernel to use with Kdump/Kexec from zero (*make allnoconfig*). Though
time consuming and a bit frustrating at times, its something that teached me
tremendously. The demanding trial and error process actually made me assimilate many key
kernel configuration symbols and how they work together, it was a great exercise.

This time, I knew that it wouldn't be as simple as it were for a x86_64 virtual machine
that emulates ancient hardware. With the Pi, I wanted to run an arm64 recent upstream
kernel with the minimal possible amount of firmware aswell. Though there are many
great [resources][rpi-kdoc] [available][elinux-kernel], I was on my own. Here is what I came up with.

## Debian Root Filesystem

First things first, we need a system to boot into. For that, I chose
[debootstrap][debootstrap] to create a full Debian system for the arm64 architecture.
This way you can chroot into the filesystem and act freely before properly burning an
image into an sd card. This is the process (all the variables in this post are
illustrative):

{% highlight bash %}
#!/bin/bash

ROOTFS_PATH="$HOME/debian/arm64"

apt install debootstrap qemu-user-static binfmt-support qemu-system-arm

debootstrap --arch arm64 \
            --components=main,contrib,non-free,non-free-firmware \
            --foreign \
            trixie \
            $ROOTFS_PATH

chroot $ROOTFS_PATH bash -c "/debootstrap/debootstrap --second-stage"
{% endhighlight %}

- qemu-user-static is [needed][deb-qemu] to actually be able to chroot into the root directory and
  execute the arm64 programs therein from a foreign host (in my case, x86_64).
- "trixie" is the current [testing][deb-releases] distribution.
- Notice the *-\-foreign* argument to debootstrap. This is needed because you're building a
root that is of an architecture that is different than your host's. Because of that, an
extra step is [required][debootstrap-docs] (the second stage).

Now, add the */etc/fstab* file while at it:

{% highlight bash %}
#!/bin/bash

echo "/dev/mmcblk0p2 / ext4 defaults,noatime 0 1" > $ROOTFS_PATH/etc/fstab
{% endhighlight %}

## Kernel build

You could build your kernel from inside the aarch64 [^2] root, but that would have a hit on
performance and also increase the size of the root with all the required packages to
make it happen. We wanna keep things nice and clean. There are many [options][crosstool]
of toolchains out there that you can use for cross-compilation. This is the Debian way:

{% highlight bash %}
#!/bin/bash

apt install gcc-aarch64-linux-gnu
{% endhighlight %}

If you haven't yet, this is the time you actually pull the Linux sources. If you're not
familiar with the [linux-next][next-docs] tree, that's where the kernel developers merge
their changes targeting the next [merge window][kdev]. Its goal is to be a reliable
preview of what the next version will look like, thus it conveniently anticipate many
common problems such as build errors and commit conflicts. Go ahead and setup your
sources, with something like this -- I can't recommend [git-worktrees][git-worktree]
highly enough:

{% highlight bash %}
#!/bin/bash

# create your local work directory
LINUX_DIR="$HOME/linux"
mkdir $LINUX_DIR
cd $LINUX_DIR

# clone a bare repository
git clone --bare -o linus git://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
cd linux.git

# add linux-next remote
git remote add next git://git.kernel.org/pub/scm/linux/kernel/git/next/linux-next.git

# update all remotes
git remote update

# add a worktree for the kernel you want to hack
git worktree add --checkout ../next-arm64 next-20231011
cd ../next-arm64
{% endhighlight %}

Now let's export some variables. When you [export][bash-docs] a variable, it is passed down to
the processes that you call, so `export ARCH=arm64; make` has the same effect as of
`ARCH=arm64 make`. This is useful if you're constantly running the same commands over
and over again and don't want a bloated sea of characters in your prompt.

{% highlight bash %}
#!/bin/bash

# the target architecture
export ARCH=arm64
# fixed part of the binutils filenames from gcc-aarch64-linux-gnu package
export CROSS_COMPILE=/usr/bin/aarch64-linux-gnu-
# where the kernel will be installed to
export INSTALL_PATH=$ROOTFS_PATH/boot
# where the modules will be installed to
export INSTALL_MOD_PATH=$ROOTFS_PATH
# where the device tree bindings of the Pi will be installed to
export INSTALL_DTBS_PATH=$ROOTFS_PATH/boot
{% endhighlight %}

All of these are properly documented [here][kbuild], apart from the usual "*make help*".
There's also some precious tips in the kernel [documentation][kconfig], I wasn't aware
of the scripts/diffconfig tool before, which is very useful for messing around with
configuration files. Also, inspecting old files with "*make listnewconfig*" can be
specially useful when analyzing vendor defaults.

Now comes the fun part, configuring the kernel:

{% highlight bash %}
#!/bin/bash

make allnoconfig
# from `make help`:
# allnoconfig     - New config where all options are answered with no

make menuconfig
# TIP: when you search for a string using '/' character, the list provided
# have numbers attached to them, which you can use to quickly jump to its
# entry in the menu; you can also use '?' character for help.
{% endhighlight %}

For starters, you need to enable support for the [chip][rpi-wiki]:

```
ARCH_BCM             Broadcom System-on-Chip systems
ARCH_BCM2835         BCM2837 and BCM2711 SoCs (Raspberry Pi 3 and 4)
```

Then, you might wanna see some output on the screen -- through the HDMI port in my case:

```
DRM                  Direct Rendering Infrastructure, introduced in XFree86 4.0
DRM_V3D              Broadcom V3D GPU
DRM_FBDEV_EMULATION  Legacy fbdev emulation for DRM
FB                   Frame buffer device abstraction layer
FB_SIMPLE            Support for a simple framebuffer
```

Later, we will burn an image into a sd card and use it for booting, so we need support
for it and also the EXT4 filesystem which will be used in the root partition:

```
MMC                  MMC/SD/SDIO I/O support
MMC_BCM2835          MultiMediaCard controller driver for the BCM2835
EXT4_FS              Extended Filesystem 4th generation
```

After successfully recognizing the disk and the root partition, the kernel will call the
*/sbin/init*, which in this case is a link to */lib/systemd/systemd* [^3]. For that, there
are many requirements:

```
BINFMT_ELF           Support for Executable and Linkable Format (ELF)
DEVTMPFS             Early ramfs instance at bootup for /dev
TMPFS                In-memory filesystem support
CGROUPS              Process grouping isolation
INOTIFY_USER         File monitoring interface for userspace
NET                  Networking support
UNIX                 Unix domain sockets
```

At this point you will land on a login prompt, but there will be no way of doing
anything because you haven't added support for input yet. Let's say you want to use a
simple USB keyboard, you first need to enable the features that permit the kernel to
power and communicate with devices:

```
MAILBOX              On-chip processors communication framework
BCM2835_MBOX         Mailbox for the BCM2835 SoC
RASPBERRYPI_FIRMWARE Support communication with the Raspberry Pi firmware
RASPBERRYPI_POWER    Raspberry Pi power domain support
PM                   Device Power Management core functionality
```

Then you can actually add the USB and HID buses:

```
USB_SUPPORT          Core Universal Serial Bus (USB) support
USB                  Enable host-side USB support
USB_DWC2             Raspberry Pi 3 DWC2 USB Controller
HID_SUPPORT          Human interface devices support
```

In my case, this is the driver I needed:

```
INPUT_KEYBOARD       Enables keyboard drivers selection submenu
KEYBOARD_ATKBD       Standard AT/PS2 keyboard
```

That's it, should be enough to have a working root. Build and install the kernel now,
making sure you have installed its [requirements][kdep]:

{% highlight bash %}
#!/bin/bash

make -j$(nproc)
# make modules_install
make dtbs_install
make install
# ...or make bindeb-pkg, of course :)
{% endhighlight %}

## Boot Loader

To boot from a sd card, you need a vfat partition at the start of the disk containing
the firmware files. This is gonna be the */boot*, which have the kernel and the [device
tree blobs][kdtb] already. You need three files from the vendor:

- [/boot/bootcode.bin][bootcode.bin]
- [/boot/start.elf][start.elf]
- [/boot/fixup.dat][fixup.dat]

So go ahead and download them into your arm64 root's */boot* directory:

{% highlight bash %}
#!/bin/bash

wget https://github.com/raspberrypi/firmware/raw/master/boot/bootcode.bin $ROOTFS_PATH/boot
wget https://github.com/raspberrypi/firmware/raw/master/boot/start.elf $ROOTFS_PATH/boot
wget https://github.com/raspberrypi/firmware/raw/master/boot/fixup.dat $ROOTFS_PATH/boot
{% endhighlight %}

Then, you need two configuration files:

- [**/boot/config.txt**][rpi-config]

```
arm_64bit=1
kernel=vmlinuz-6.6.0-rc5-next-20231011
device_tree=/broadcom/bcm2837-rpi-3-b.dtb
```

- **/boot/cmdline.txt**

```
root=/dev/mmcblk0p2 rootfstype=ext4 rootwait
```

## Image Creation

Now that you have a fully functional root filesystem, it's time to prepare it to go into
the sd card so that the Pi can finally boot it:

{% highlight bash %}
#!/bin/bash

apt install dosfstools  # mkfs.vfat

dd if=/dev/zero of=arm64.img bs=1M count=1024

losetup -fP arm64.img
losetup -a  # note which /dev/loopN is used, in my case N=0

cfdisk /dev/loop0
# DOS partition scheme:
#   1: 128M, type: b (W95 FAT32), bootable
#   2: ++, type: 83 (Linux)

mkfs.vfat /dev/loop0p1
mkfs.ext4 /dev/loop0p2

tmp=$(mktemp -d)
mount /dev/loop0p2 $tmp
mount /dev/loop0p1 $tmp/boot

cp -a $ROOTFS_PATH/. $tmp/.

umount $tmp/boot
umount $tmp

sync

losetup -d /dev/loop0

# replace <SD_CARD> with the correct value on your system
dd if=arm64.img of=/dev/<SD_CARD> status=progress
{% endhighlight %}

## What's next?

Well, there are many things to explore if you want to keep tinkering with the kernel
configuration symbols. For example, you could decide to add the */dev/mmcblk0p1* boot
partition to your */etc/fstab*, but that requires VFAT support:

```
VFAT_FS              VFAT (Windows-95) fs support
NLS_CODEPAGE_437     Default DOS codepage
NLS_ISO8859_1        ISO/IEC 8859-1 character set
```

Maybe you want to figure out what's needed for complete networking support, or
learn more about [GPIO][elinux-gpio].

## Conclusion

Knowing exactly what you're enabling on the kernel and being able to actually boot it
can give you important insight about the different areas of knowledge that spans this
vast project. You'll be able to quickly find your way into whatever feature that you
might wanna learn about more. Although a nice exercise, you're probably better off with
the [vendor configuration][rpi-kconfig] as a starting point. There will be many things
there that you're probably not gonna use, but most of what you need will also be there.
Overall, this is one of the trade-offs distro kernel teams have to deal with.

If you have any question, comment, critic, suggestion or if you spotted an error in
here, please don't hesitate in reaching out to me at [ricardo@marliere.net][email].
Good luck and happy hacking!

[^1]: Funtoo is a Gentoo fork made by Gentoo's creator, Daniel Robbins.
[^2]: [AArch64][arm-docs] is the same as ARM64.
[^3]: lib/systemd/systemd: ELF 64-bit LSB pie executable, ARM aarch64, version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux-aarch64.so.1, (...)
[better-workflow]: {% link _posts/2023-10-08-better-workflow.markdown %}

[next]: https://git.kernel.org/pub/scm/linux/kernel/git/next/linux-next.git/
[funtoo]: https://www.funtoo.org
[sysroot]: https://github.com/rbmarliere/sysroot
[rpi-kernel]: https://github.com/raspberrypi/linux
[debootstrap]: https://tracker.debian.org/pkg/debootstrap
[deb-releases]: https://www.debian.org/releases/
[crosstool]: https://kernel.org/pub/tools/crosstool/
[bash-docs]: https://www.gnu.org/software/bash/manual/bash.html#Bourne-Shell-Builtins
[elinux-kernel]: https://elinux.org/RPi_Upstream_Kernel_Compilation
[rpi-kconfig]: https://raw.githubusercontent.com/raspberrypi/linux/rpi-6.1.y/arch/arm64/configs/bcm2711_defconfig
[arm-docs]: https://developer.arm.com/documentation/ddi0487/latest/
[debootstrap-docs]: https://sources.debian.org/src/debootstrap/1.0.132/debootstrap.8/#L126
[deb-qemu]: https://wiki.debian.org/EmDebian/CrossDebootstrap#QEMU.2Fdebootstrap_approach
[next-docs]: https://www.kernel.org/doc/man-pages/linux-next.html
[kdev]: https://www.kernel.org/doc/html/latest/process/2.Process.html
[git-worktree]: https://git-scm.com/docs/git-worktree
[kbuild]: https://www.kernel.org/doc/html/latest/kbuild/kbuild.html
[email]: mailto:ricardo@marliere.net
[kdep]: https://www.kernel.org/doc/html/latest/process/changes.html
[kconfig]: https://www.kernel.org/doc/html/latest/kbuild/kconfig.html
[kdtb]: https://www.kernel.org/doc/html/latest/devicetree/usage-model.html
[bootcode.bin]: https://github.com/raspberrypi/firmware/raw/master/boot/bootcode.bin
[start.elf]: https://github.com/raspberrypi/firmware/raw/master/boot/start.elf
[fixup.dat]: https://github.com/raspberrypi/firmware/raw/master/boot/fixup.dat
[rpi-config]: https://www.raspberrypi.com/documentation/computers/config_txt.html
[rpi-kdoc]: https://www.raspberrypi.com/documentation/computers/linux_kernel.html
[rpi-wiki]: https://en.wikipedia.org/wiki/Raspberry_Pi#Model_comparison
[elinux-gpio]: https://elinux.org/RPi_Low-level_peripherals
