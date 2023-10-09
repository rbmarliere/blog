---
layout: post
title: "A better workflow for kernel debugging"
date: 2023-10-08 15:15:00 -0300
tags: linux kernel mentorship
---

If you followed along my last [post][simple-workflow], you may have have found some
difficulties like I did. Namely, making sure the work is done without confusion. In this
post I want to share a few tips that helped me stay organized and effective, I hope its
helpful to some of you as well.

When I started investigating bugs I used to have a single *git* tree and then every time
I wanted to switch bugs or build a stable kernel it was *.config* hell, a complete mess.
I tried, then, maintaining different local repositories for different trees. But this is
a disk waste. Then I learned about the "*make O=*" trick to build the objects somewhere
else. It helped but I still had to checkout different commits and keep a tab of what I
was doing in my head. The solution was of course to use [git-worktree][git-worktree].
This is the process (all the variables in this post are illustrative):

{% highlight bash %}
#!/bin/bash

# create your local work directory
LINUX_DIR="$HOME/linux"
mkdir $LINUX_DIR
cd $LINUX_DIR

# clone a bare repository
git clone --bare -o linus git://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
cd linux.git

# add other relevant remotes to you
git remote add shuah git://git.kernel.org/pub/scm/linux/kernel/git/shuah/linux-kselftest.git
git remote add staging git://git.kernel.org/pub/scm/linux/kernel/git/gregkh/staging.git
git remote add stable git://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git
# (...)

# update all the remotes
git remote update
{% endhighlight %}

Now that you have a bare repository, you can add worktrees to split the different
things you might be doing into different directories. Here's an example:

{% highlight bash %}
#!/bin/bash

cd $LINUX_DIR/linux.git

# add a worktree to track the stable/linux-6.5.y branch
git worktree add --track --branch linux-6.5.y ../stable stable/linux-6.5.y

# add a worktree for a syzkaller bug
git worktree add --checkout ../845cd8e5c47f2a125683 69b41ac87e4a
{% endhighlight %}

This way, you can have different directories with different kernel *.config* files and
different *git* histories and patches you might be working on. In the example above,
[845cd8e5c47f2a125683][845cd8e5c47f2a125683] is a random bug I selected and [69b41ac87e4a][69b41ac87e4a]
is the commit ref. from the *2023/01/04 06:51* row of its crashes. You would proceed by
downloading its *.config* and other relevant files such as the reproducers. For that
it's best to have a directory only for these:

{% highlight bash %}
#!/bin/bash

BUGS_DIR="$HOME/syzkaller-bugs"
mkdir -p $BUGS_DIR/845cd8e5c47f2a125683
cd $BUGS_DIR/845cd8e5c47f2a125683

# download relevant files for the bug
# (you might wanna split those into timestamped directories,
#   e.g. $BUGS_DIR/845cd8e5c47f2a125683/202301040651 )
wget "https://syzkaller.appspot.com/text?tag=KernelConfig&x=9babfdc3dd4772d0" -O .config
wget "https://syzkaller.appspot.com/text?tag=ReproC&x=142a1eaa480000" -O rep.c
wget "https://storage.googleapis.com/syzbot-assets/d4a7091814ba/bzImage-69b41ac8.xz"
unxz bzImage-69b41ac8.xz

# make your life easier with a link to the worktree
ln -s $LINUX_DIR/845cd8e5c47f2a125683 linux

cp .config linux/
{% endhighlight %}

After the usual building process of the bzImage, its time to jump into a virtual machine
to test your changes. This can also be chaotic and inspired by this great [mentorship
session][tricks] done by Joel Fernandes, I wrote some [scripts][rq] to make my life
easier. The main tool is *rq*:

```
$ rq -h

usage: rq [-h] [-I IMG_DIR] [-k KERNEL] [--no-kernel] [-d DISK] [--no-disk]
          [-i INITRD] [-b BIOS] [-p PORT] [-f [F ...]] [-F [F ...]] [-c]
          [-a APPEND] [-g] [--wait-gdb] [--no-snapshot] [--debug]

Run qemu with sane defaults

options:
  -h, --help            show this help message and exit
  -I IMG_DIR, --img-dir IMG_DIR
                        directory containing images (e.g. initramfs, disk,
                        bios), can be set through RQ_IMGDIR environment variable
                        (default: /home/rbmarliere/images)
  -k KERNEL, --kernel KERNEL
                        kernel bootable image to use with -kernel (default:
                        linux/arch/x86/boot/bzImage)
  --no-kernel           do not use -kernel (default: False)
  -d DISK, --disk DISK  disk image to use with -drive (default:
                        /home/rbmarliere/images/disk.img)
  --no-disk             do not use -drive (default: False)
  -i INITRD, --initrd INITRD
                        initrd image to use with -initrd (default: None)
  -b BIOS, --bios BIOS  bios image to use with -bios (default:
                        /home/rbmarliere/images/seabios.bin)
  -p PORT, --port PORT  host's port to forward into guest's ssh port using -net
                        (default: 10022)
  -f [F ...]            file(s) to copy into the disk (default: None)
  -F [F ...]            file(s) to copy into the cpio initrd (default: None)
  -c, --crash           append crashkernel= to cmdline (default: False)
  -a APPEND, --append APPEND
                        append kernel parameters (default: None)
  -g, --gdb             use -s (default: False)
  --wait-gdb            use -S if -s is used (default: False)
  --no-snapshot         do not use -snapshot (default: False)
  --debug               print qemu command instead of running it (default:
                        False)
```

This makes me able to do many things without having to worry about searching back my
shell history for the right *qemu* command. It's not perfect, but it helps. If you
noticed, the *--bios* parameter defaults to */home/rbmarliere/images/seabios.bin*, this
is because I'm using a custom built image with a [patch][seabios_patch] to
[SeaBIOS][seabios] that gets rid of the code that truncates long lines in the terminal,
so that I don't lose precious info from logs. This is what it calls on a default run
without parameters, inside a directory such as *$BUGS_DIR/845cd8e5c47f2a125683*:

```
$ rq --debug

qemu-system-x86_64 \
    -m 8G \
    -smp 2,sockets=2,cores=1 \
    -net nic,model=e1000 \
    -net user,host=10.0.2.25,hostfwd=tcp::10022-:22 \
    -enable-kvm \
    -nographic \
    -bios /home/rbmarliere/images/seabios.bin \
    -snapshot \
    -kernel linux/arch/x86/boot/bzImage \
    -drive format=raw,file=/home/rbmarliere/images/disk.img
```

If you're feeling courageous and want to give it a try, install them with something like
this:

{% highlight bash %}
#!/bin/bash

GIT_DIR="$HOME/git"

git clone git@github.com:rbmarliere/rq.git $GIT_DIR/rq

cd $HOME/.local/bin
for file in $GIT_DIR/rq/*; do ln -s $file; done

# add ~/.local/bin to your $PATH, if you don't have it
echo export PATH="$PATH:$HOME/.local/bin" >> ~/.bashrc
{% endhighlight %}

Some examples:

{% highlight bash %}
#!/bin/bash

export RQ_IMGDIR=~/images

# run a specific kernel after copying a reproducer into the disk
rq -k bzImage-69b41ac8 -f rep.c

# use an initramfs instead of a disk as the root filesystem
rq -i initramfs.cpio.gz -F rep.c --no-disk -a "root=/dev/ram"

# make permanent changes to the disk root filesystem
rq --no-snapshot

# debug the VM with gdb
rq -g --wait-gdb

{% endhighlight %}

## Kdump

Recently I was looking into a bug that is a concurrency problem. That means that,
although the reproducer provided can in fact reproduce the problem, it happens at random
times. That means you are probably gonna have to wait for a long time. So as much as the
*gdb* workflow presented in the [last post][simple-workflow] is very useful, for bugs
like this it can be quite painful. Using [Kdump][kdump-doc] and the [crash utility][crash] enabled me to inspect the system state at any time without having to depend on a running virtual machine.

I went through this [excellent presentation][postmortem] by Steven Rostedt and I must
say, it took me a while to grasp the process and setup everything. But its a very good
tool to have, sometimes. Hopefully I can help you save some time. The first thing you
need is to build a "crash" kernel. This is the kernel that it's going to replace the
system kernel after a panic. Here is what I did:

{% highlight bash %}
#!/bin/bash

cd $LINUX_DIR/linux.git

git worktree add --checkout ../crash v6.5.6
cd ../crash

make allnoconfig
# from `make help`:
# allnoconfig     - New config where all options are answered with no

make menuconfig
# TIP: when you search for a string using '/' character, the list provided
# have numbers attached to them, which you can use to quickly jump to its
# entry in the menu; you can also use '?' character for help.

# search and enable the following:

# 64BIT               64-bit kernel
# PCI                 support for PCI bus
# ATA                 support for ATA hard disk used in -disk parameter
# ATA_PIIX            ATA controller emulated by the default qemu x86_64 machine
# SERIAL_8250         support for serial ports
# SERIAL_8250_CONSOLE this enables the virtual machine console to be readable

# EXT4_FS             the filesystem used before, when creating the root fs
# BLK_DEV_SD          this makes /dev/sda visible

# NET                 we want networking inside the VM
# NETDEVICES          add support for networking devices
# E1000               driver for the -net nic,model=e1000 parameter

# INET                TCP/IP support
# UNIX                Unix sockets support, e.g. for /var/log/* and system utils
# PACKET              Packet protocol

# BINFMT_ELF          enables kernel to execute ELF files
# BINFMT_SCRIPT       enables kernel to execute '#!' scripts
# DEVTMPFS            create a /dev early on, to support early API filesystems
# DEVTMPFS_MOUNT      auto mount /dev
# TMPFS               support early API filesystems

# CRASH_DUMP          Kdump requirement
# RELOCATABLE         Kdump requirement

# CGROUPS             * (systemd requirement)
# INOTIFY_USER        * (systemd requirement)
# DEBUG_FS            *
# SECURITYFS          *
# CONFIGFS_FS         *
# BINFMT_MISC         *
# AUTOFS_FS           * (for proc-sys-fs-binfmt_misc.automount service)

make bzImage
{% endhighlight %}

This was the first time I configured a minimal kernel from zero and I learned a lot from
it. These are the symbols that need to be enabled to successfully boot the kernel after
a crash. Note that the ones marked with **\*** are dependencies for the root filesystem.
Systemd needs CGROUPS and INOTIFY_USER and the others come from the */etc/fstab* written
by the syzkaller's [create-image.sh][syzkaller-fstab] script that I used to create my
root filesystem (therefore, optional).

After this, you should install *kdump-tools* inside your VM:

{% highlight bash %}
#!/bin/bash

# you can use any kernel to boot the VM, e.g.
cd $BUGS_DIR/845cd8e5c47f2a125683
rq --no-snapshot -k bzImage-69b41ac8

# after booting, inside the VM
apt install kdump-tools
vi /etc/default/kdump-tools
{% endhighlight %}

Edit the */etc/default/kdump-tools* file, inside the VM, accordingly (remember to use
*-\-no-snapshot* to make it permanent). Please refer to Debian's documentation on
kdump-tools in [/usr/share/doc/kdump-tools/README.Debian][kdump-deb-doc]. Here is mine:

```
USE_KDUMP=1
KDUMP_COREDIR="/var/crash"
SSH="rbmarliere@10.0.2.25"
SSH_KEY="/root/.ssh/id_rsa"
```

This way, whenever a crash happens, the *kdump-tools* will transfer the dump file into my
host's */var/crash/* directory. Before, I did the usual *ssh* authorization process of
creating a *ssh* key with [*ssh-keygen*][ssh-keygen] in the host, transferring the private
id_rsa into the VM and pasting the public key into the host's *~/.ssh/authorized_keys*.
Also, make sure */var/crash* is owned and writable by your user in the host, or change
*KDUMP_COREDIR* to something else.

Now we are only missing three things:

- Boot the system kernel with *crashkernel=* cmdline;
- Load the crash kernel into memory using *kexec*;
- Trigger the crash;

{% highlight bash %}
#!/bin/bash

# make your life easier with a link to the crashkernel
cd $BUGS_DIR/
ln -s $LINUX_DIR/crash/arch/x86_64/boot/bzImage crashkernel

# copy the crashkernel into the root filesystem next time you boot a VM
# -f copies the file into /root/ by default
# -c will append crashkernel=256M to the boot cmdline
cd $BUGS_DIR/845cd8e5c47f2a125683
rq -c -f ../crashkernel

# after boot, inside the VM
kexec -p /root/crashkernel --append="console=ttyS0 root=/dev/sda"

# trigger the crash with CONFIG_MAGIC_SYSRQ=y
echo c > /proc/sysrq-trigger

# ...or run a syz reproducer
gcc rep.c && ./a.out
{% endhighlight %}

Note the need for *root=/dev/sda* for the automatic handling of the crash dump taking
place. If all went well, you're gonna see the dump and the dmesg files under your host
*/var/crash* directory. But how to use it? Here enters the [crash utility][crash]. For
some reason that I didn't fully understand -- probably Debian related[^1] -- I was only able
to inspect the dump file using a locally built binary. Therefore, don't install the
[crash][crash-deb] package in your host.

{% highlight bash %}
#!/bin/bash

$GIT_DIR/crash/crash $LINUX_DIR/845cd8e5c47f2a125683/vmlinux dump.202310081631
{% endhighlight %}

```
crash 8.0.3
Copyright (C) 2002-2022  Red Hat, Inc.
Copyright (C) 2004, 2005, 2006, 2010  IBM Corporation
Copyright (C) 1999-2006  Hewlett-Packard Co
Copyright (C) 2005, 2006, 2011, 2012  Fujitsu Limited
Copyright (C) 2006, 2007  VA Linux Systems Japan K.K.
Copyright (C) 2005, 2011, 2020-2022  NEC Corporation
Copyright (C) 1999, 2002, 2007  Silicon Graphics, Inc.
Copyright (C) 1999, 2000, 2001, 2002  Mission Critical Linux, Inc.
Copyright (C) 2015, 2021  VMware, Inc.
This program is free software, covered by the GNU General Public License,
and you are welcome to change it and/or distribute copies of it under
certain conditions.  Enter "help copying" to see the conditions.
This program has absolutely no warranty.  Enter "help warranty" for details.

GNU gdb (GDB) 10.2
Copyright (C) 2021 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Type "show copying" and "show warranty" for details.
This GDB was configured as "x86_64-pc-linux-gnu".
Type "show configuration" for configuration details.
Find the GDB manual and other documentation resources online at:
    <http://www.gnu.org/software/gdb/documentation/>.

For help, type "help".
Type "apropos word" to search for commands related to "word"...

please wait... (determining panic task)
WARNING: active task ffff888102323780 on cpu 1 not found in PID hash

      KERNEL: /mnt/md0/linux/845cd8e5c47f2a125683/vmlinux
    DUMPFILE: dump.202310081631  [PARTIAL DUMP]
        CPUS: 2
        DATE: Sun Oct  8 13:31:48 -03 2023
      UPTIME: 00:01:22
LOAD AVERAGE: 0.95, 0.47, 0.17
       TASKS: 3
    NODENAME: syzkaller
     RELEASE: v6.2-rc3~30
     VERSION: #7 SMP PREEMPT_DYNAMIC Sun Oct  8 10:34:01 -03 2023
     MACHINE: x86_64  (3892 Mhz)
      MEMORY: 8 GB
       PANIC: "Kernel panic - not syncing: sysrq triggered crash"
         PID: 4194
     COMMAND: "bash"
        TASK: ffff888102323780  [THREAD_INFO: ffff888102323780]
         CPU: 1
       STATE: TASK_RUNNING (PANIC)

crash> bt
PID: 4194     TASK: ffff888102323780  CPU: 1    COMMAND: "bash"
 #0 [ffffc9000166f968] machine_kexec at ffffffffb0728db3
 #1 [ffffc9000166fa58] __crash_kexec at ffffffffb0a417a6
 #2 [ffffc9000166fbb8] panic at ffffffffb077eedd
 #3 [ffffc9000166fca0] sysrq_handle_crash at ffffffffb1e3228c
 #4 [ffffc9000166fcb0] __handle_sysrq at ffffffffb1e334e0
 #5 [ffffc9000166fcf8] write_sysrq_trigger at ffffffffb1e34ea4
 #6 [ffffc9000166fd18] proc_reg_write at ffffffffb122c99c
 #7 [ffffc9000166fd58] vfs_write at ffffffffb10324d4
 #8 [ffffc9000166fda8] vfs_fstatat at ffffffffb104cac2
 #9 [ffffc9000166fde0] __do_sys_newfstatat at ffffffffb104d252
#10 [ffffc9000166ff38] do_syscall_64 at ffffffffb5aab7c9
#11 [ffffc9000166ff50] entry_SYSCALL_64_after_hwframe at ffffffffb5c0008b
    RIP: 00007fb4c359cb00  RSP: 00007fff42199c08  RFLAGS: 00000202
    RAX: ffffffffffffffda  RBX: 0000000000000002  RCX: 00007fb4c359cb00
    RDX: 0000000000000002  RSI: 00005645a39eeaa0  RDI: 0000000000000001
    RBP: 00005645a39eeaa0   R8: 0000000000000400   R9: 0000000000000410
    R10: 0000000000000000  R11: 0000000000000202  R12: 0000000000000002
    R13: 00007fb4c3679780  R14: 0000000000000002  R15: 00007fb4c3674d00
    ORIG_RAX: 0000000000000001  CS: 0033  SS: 002b
crash> info threads
  Id   Target Id         Frame
* 1    CPU 0             <unavailable> in ?? ()
  2    CPU 1             <unavailable> in ?? ()
crash> panic
panic = $1 =
 {void (const char *, ...)} 0xffffffffb077e9b0 <panic>
crash> ps
      PID    PPID  CPU       TASK        ST  %MEM      VSZ      RSS  COMM
>       0       0   0  ffffffffb722b800  RU   0.0        0        0  [swapper/0]
        0       0   1  ffff888102e61bc0  RU   0.0        0        0  [swapper/1]
>    4194      -1   1  ffff888102323780  RU   0.0     4772     3924  bash
```
--- *Note that you should have DEBUG_INFO enabled on the system kernel that crashed!*

## General tips

I use this as my */root/.bash_profile* inside the VM:

{% highlight bash %}
# fix the row/col height of the terminal
stty rows 71 cols 113

# auto build whatever rep.c copied into /root/ using `rq -f`
[ -f ~/rep.c ] && gcc ~/rep.c

# make your life easier with some aliases
alias a="./a.out"
alias s="./syz-execprog syz"
alias kex="kexec -p crashkernel --append='console=ttyS0 root=/dev/sda'"
{% endhighlight %}

Also this *~/.ssh/config* in my host system to quickly *ssh* or *scp* into the VM:

```
Host syz
	Hostname localhost
	User root
	NoHostAuthenticationForLocalhost yes
	IdentityFile ~/images/syzkaller.id_rsa
	Port 10022
```

In my host *~/.bashrc*, I use these:
{% highlight bash %}
LINUX_DIR=~/linux

# function to get maintainer for a patch from anywhere
get_maintainer() {
	if [ -z "$1" ]; then
		echo "usage: get_maintainer <patch>"
		return
	fi
	patch=$(realpath "$1")
	pushd $LINUX_DIR/linus > /dev/null  # assumes a 'linus' worktree there
	scripts/get_maintainer.pl "$patch"
	popd > /dev/null
}

# enables the use of `cd bugs` from anywhere
shopt -s cdable_vars
bugs=~/syzkaller-bugs

# a few useful aliases
alias cdL="cd $LINUX_DIR/linux/linux.git"
alias cdl="cd $LINUX_DIR/linux"
alias checkpatch="$LINUX_DIR/linus/scripts/checkpatch.pl"
alias db="gdb -tui -ex 'target remote :1234' -d linux"
alias make='make -j$(nproc)'
{% endhighlight %}

I hope I provided some inspiration to improve your workflow if you are a kernel hacker
wannabe like myself. If you have any question, comment, critic, suggestion or if you
spotted an error in here, please don't hesitate in reaching out to me at
[*ricardo@marliere.net*][email].


[^1]: As far as I can tell, *kdump-tools* is packaged with the clear use case of dumping a running Debian kernel and depends on the distro */boot/\** files
[simple-workflow]: {% link _posts/2023-09-06-simple-workflow.markdown %}

[git-worktree]: https://git-scm.com/docs/git-worktree
[845cd8e5c47f2a125683]: https://syzkaller.appspot.com/bug?extid=845cd8e5c47f2a125683
[69b41ac87e4a]: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=69b41ac87e4a664de78a395ff97166f0b2943210
[rq]: https://github.com/rbmarliere/rq
[email]: mailto:ricardo@marliere.net
[seabios]: https://gitlab.com/qemu-project/seabios
[seabios_patch]: https://github.com/cirosantilli/linux-kernel-module-cheat/issues/110#issuecomment-1498600775
[crash]: https://github.com/crash-utility/crash
[kdump-doc]: https://www.kernel.org/doc/html/latest/admin-guide/kdump/kdump.html
[postmortem]: https://www.youtube.com/watch?v=aUGNDJPpUUg
[tricks]: https://www.linuxfoundation.org/webinars/linux-kernel-debugging-tricks-of-the-trade
[syzkaller-fstab]: https://github.com/google/syzkaller/blob/master/tools/create-image.sh#L161
[kdump-deb-doc]: https://salsa.debian.org/debian/kdump-tools/-/blob/master/debian/kdump-tools.README.Debian
[ssh-keygen]: https://man.openbsd.org/ssh-keygen.1
[crash-deb]: https://tracker.debian.org/pkg/crash
