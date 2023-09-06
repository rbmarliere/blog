---
layout: post
title: "A simple workflow to debug the Linux Kernel"
date: 2023-09-06 15:45:00 -0300
tags: linux kernel mentorship
---

Hello again!

I'm happy to announce that I have been accepted as a mentee for the 2023 Fall edition of
the ["Linux Kernel Bug Fixing"][mentorship] mentorship from the Linux Foundation.
Writing about the experience is one of the required tasks and that is indeed the
motivation behind the creation of this blog, although I flirted with the idea before but
never got to actually publish anything. Since the main task of the mentorship is to fix
bugs, I'm gonna try to organize my thoughts about the workflow for accomplishing that in
this post. The basic assumption is of course x86_64, for the sake of simplicity.

The Linux Kernel is a vast project that spans many areas of computing and long are the
days where a single mind could understand it all as a whole. The majority of the work in
it is done by big companies that need to support it for their operation. It can be
overwhelming to even try finding a [place where to start][dev-proc] and this is why intern and
mentorship programs are so important! It is typical for first time contributors to any FOSS
project to focus on QA - Quality Assurance. That could mean many things but in the
context of the Linux Kernel, fixing bugs is the best example. In other projects, simply
using it extensively and reporting a bug found is already a great contribution, but for
simple desktop users such as myself it is rather difficult to come across a bug in the
kernel, these days.

This is where automated tools such as [Syzkaller][syzkaller] can come in handy. It is a
fuzzer and nowadays it supports all other major kernels. Software fuzzing is a technique
that feeds random inputs into a program until it crashes and it can be done with pretty
much any program that takes input somehow. In the context of the Linux Kernel, that can
get quite complicated, but it is not the scope of this text to go much into these
details. Please refer to [this excellent presentation][dmitry] by one of the original
developer of the tool, Dmitry Vyukov. For the purpose of this text it is sufficient to
say that there is [infrastructure][syzbot] setup that extensively test the kernel in
many [trees][syz-trees] and architectures, so that bugs get reported to the mailing
lists as they're found.

There are many other tools available for testing and finding bugs in the kernel, of
which Sergio Prado wrote a [nice summary][prado1] about. But the goal here is to focus
on the *fixing* part of the process. For that I must refer the reader to [Chapter
4][ldd4] of the classic book "Linux Device Drivers". It is also a good idea to read the
available documentation of the kernel on the subject, such as [this][kernel-dbg].
Another technique which is essential to the task is tracing, which was masterfully
introduced in [this talk][tracing] by Steven Rostedt.

Another [great introduction][prado2] on debugging tools and techniques was made by
Sergio Prado. In it, he starts by splitting the most common problems in 5 categories,
namely:
-	Crash
-	Lockup / Hang
-	Logic / Implementation
-	Resource Leakage
-	Performance

Each one of these require a different set of skills and strategies, which include log
and dump analysis, tracing and profiling, interactive debugging, etc. The basic workflow
to fix a bug is comprised of a few high level steps:
-	Find the bug
-	Reproduce it
-	Identify the cause
-	Develop the fix
-	Test that it worked

For each of these steps there are several sub-steps, depending on the degree of
complexity of the problem. For simplicity, let's illustrate what a common crash analysis
reported by Syzkaller would look like. First, go to its dashboard and find a bug you
might be interested on. Make sure to read [their documentation][syz-docs] before
that. You gonna need to know the basics of a kernel *oops*, too. Refer to [here][oops]
as a good starting point.

After finding the bug, you must be able to reproduce it. Many of the bugs reported have
C programs carefully crafted by the tool that will trigger the issue. There are also a
special *syz* program, but we'll get into that later. For each bug, the syzbot also
supply us with the kernel *.config* file and the specific commit which triggers the issue.
That is, if you run the C reproducer on any other kernel version, or any other
configuration, the problem may not arise. So, if you want to be able to successfully
reproduce it, you must take that into account. The bot also provides a bootable
QEMU-enabled disk image and the kernel *vmlinux* and *bzImage* for your convenience (more on
that [here][syz-assets]). That is excellent for rapid testing, however if you are to
locally test a patch you're developing, you gonna need to build it anyway.

Alright, so the goal is to reproduce the error in a QEMU virtual machine with a locally
built *bzImage*. But first things first. A kernel image by itself isn't of much use
without a root filesystem from where you can run a reproducer program. I recommend at
least once going through the process of [manually creating][initrd] an initramfs to boot
into, after reading about it in the [kernel documentation][kernel-init]. Luckily, there
are many tools to accomplish that automatically, such as [Buildroot][buildroot] - which
is extra useful if you're hacking around with embedded devices and cross-compilation.
But since the context here is Syzkaller, let's use [their script][syz-img], so download
it somewhere into your system. It is an automated tool to create a suitable image to use
with QEMU. It leverages Debian's [Debootstrap][debootstrap] tool, so if you're not
running Debian this is another reason to :)

Create a trixie (current Debian's [*testing* distribution][deb-releases]) image like so
(make sure you have the required dependencies installed):

{% highlight bash %}
#!/bin/bash
wget "https://raw.githubusercontent.com/google/syzkaller/master/tools/create-image.sh"
chmod +x create-image.sh
./create-image.sh --distribuition trixie
{% endhighlight %}

This yields three files: *trixie.img*, *trixie.id_rsa* and *trixie.id_rsa.pub*, which
we'll use shortly. Now that we have the root filesystem, we need to build the kernel.
Let's take a look at [this bug][syz-bug]:

```
Hello,

syzbot found the following issue on:

HEAD commit: b84acc11b1c9 Merge tag 'fbdev-for-6.6-rc1' of git://git.ke..
git tree: upstream
console+strace: https://syzkaller.appspot.com/x/log.txt?x=10e9af87a80000
kernel config: https://syzkaller.appspot.com/x/.config?x=3aba740d8a88ff1d
dashboard link: https://syzkaller.appspot.com/bug?extid=c063a4e176681d2e0380
compiler: gcc (Debian 12.2.0-14) 12.2.0, GNU ld (GNU Binutils for Debian) 2.40
syz repro: https://syzkaller.appspot.com/x/repro.syz?x=16e4acdba80000
C reproducer: https://syzkaller.appspot.com/x/repro.c?x=14eb56dba80000

Downloadable assets:
disk image: https://storage.googleapis.com/syzbot-assets/8b5634407855/disk-b84acc11.raw.xz
vmlinux: https://storage.googleapis.com/syzbot-assets/31f561af0e06/vmlinux-b84acc11.xz
kernel image: https://storage.googleapis.com/syzbot-assets/37275212826f/bzImage-b84acc11.xz

IMPORTANT: if you fix the issue, please add the following tag to the commit:
Reported-by: syzbot+c063a4...@syzkaller.appspotmail.com

(...)
```

Note however that these resources are only available until the bugs are fixed, so if you
can't download these files at the time of reading just proceed to investigate a more
recent bug.

Let's build the kernel, so if you haven't already this is the time to clone Linus' tree:

{% highlight bash %}
#!/bin/bash
git clone git://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
cd linux
git checkout b84acc11b1c9
wget "https://syzkaller.appspot.com/x/.config?x=3aba740d8a88ff1d" -O .config
make menuconfig
# Kernel hacking
#   -> Generic Kernel Debugging Instruments
#     -> KGDB: kernel debugger
#        y
make bzImage
{% endhighlight %}

As you see, we checked out the same commit that syzbot used, under the same tree. We
also used the same *.config* file to build the *bzImage*. Note that the assumption here is
that your host is running Debian as well, which is the same system as the bot. In some
builds it uses clang so the appropriated *CC=clang* flag should be used. Also, you could
add a few debugging configurations to the kernel, since we'll be using that *bzImage* to
debug and investigate the root cause of the reported bug.

Now that we have the *bzImage*, let's use it to boot the *trixie.img* rootfs:

{% highlight bash %}
#!/bin/bash
ROOTFS="$HOME/trixie.img"
BZIMAGE="$HOME/linux/arch/x86/boot/bzImage"
qemu-system-x86_64 \
	-m 2G \
	-smp 2,sockets=2,cores=1 \
	-net nic,model=e1000 \
	-net user,host=10.0.2.25,hostfwd=tcp::10022-:22 \
	-enable-kvm \
	-nographic \
	-machine pc-q35-7.1 \
	-snapshot \
	-append "root=/dev/sda console=ttyS0 earlyprintk=serial net.ifnames=0" \
	-drive format=raw,file="$ROOTFS" \
	-kernel "$BZIMAGE"
{% endhighlight %}

This QEMU command will spawn a x86_64 virtual machine (of type pc-q35-7.1,
*qemu-system-x86_64 -machine help* for more options) and boot your kernel into the
trixie rootfs, make sure to point to the correct paths in the *$ROOTFS* and *$BZIMAGE*
variables. This command is given as a reference in the Syzkaller's
[documentation][syz-qemu], however you should take a look at QEMU's [manual
pages][qemu-doc] if you haven't yet. You can see we passed both the kernel image and the
rootfs disk as parameters, and since the *create-image.sh* script also take care of
setting up networking and a SSH server for us (that's why it also yields *trixie.id_rsa*
and *trixie.id_rsa.pub*), the command will spawn a network interface with your local
host 10022 port being forwarded into its 22 port through the *-net* argument. That means
you can log into the guest system and transfer files through the ssh protocol using
*localhost:10022*.

Another important parameter is *-snapshot*. Using it means that whatever modification
you do to the rootfs won't be persisted through reboots. So if you transfer a file into
it and halt the system, it won't be there when you start a machine again with it. That
makes possible for many guests running in parallel using the same rootfs. You could also
remove this argument for one time in order to make changes such as changing the root
password or installing a dependency. For example, if you want to reproduce bugs using
the *syz* program reproducer you're gonna need to build syzkaller locally and transfer
the *bin/linux_amd64/syz-executor* and *bin/linux_amd64/syz-execprog* files into the rootfs,
so you can transfer these files just once, without having to redo this step every time
you boot a kernel.

So now that the machine is running, you can go ahead and reproduce the bug in it. First,
download the reproducer program, then transfer it into the rootfs and finally log into
it through ssh (note that you could do this from inside the rootfs, by ssh'ing first):

{% highlight bash %}
#!/bin/bash
wget "https://syzkaller.appspot.com/x/repro.c?x=14eb56dba80000" -O reproducer.c
IDENTITY="$HOME/trixie.img.id_rsa"
scp -i "$IDENTITY" -P 10022 reproducer.c root@localhost:~/
ssh -i "$IDENTITY" -p 10022 root@localhost
{% endhighlight %}

Now, inside the virtual machine simply build the program and run it:
{% highlight bash %}
#!/bin/bash
gcc -Wall reproducer.c -o reproducer
./reproducer
{% endhighlight %}

To make things easier, I always like to add aliases to my *~/.bashrc* whenever possible.
They allow me to quickly go through the work without having to deal with the shell
history or copy/pasting. Consider these:

{% highlight bash %}
#!/bin/bash
alias qemu-='qemu-system-x86_64 -m 2G -smp 2,sockets=2,cores=1 -net nic,model=e1000 -net user,host=10.0.2.25,hostfwd=tcp::10022-:22 -enable-kvm -nographic -machine pc-q35-7.1 -append "root=/dev/sda console=ttyS0 earlyprintk=serial net.ifnames=0" -snapshot -drive format=raw,file=/home/rbmarliere/images/syzkaller/trixie.img -kernel'
alias ssh-='ssh -i /home/rbmarliere/images/syzkaller/trixie.id_rsa -p 10022 root@localhost'
alias scp-='scp -i /home/rbmarliere/images/syzkaller/trixie.id_rsa -P 10022 '
{% endhighlight %}

Lastly, to be able to really debug and step through the code with [*gdb*][gdb-docs] you
can use the *-s* argument to the QEMU command. That's when a *vmlinux* image with
debugging information is useful. Could be the one you downloaded from the syzbot bug
report or the one you built before (if you enabled KGDB as suggested, you're gonna be
better equipped than using the image that syzbot provided). Start the virtual machine
like before , appending the *-s* argument. Then, start *gdb* passing the *vmlinux* as
argument:

{% highlight bash %}
#!/bin/bash
VMLINUX="$HOME/linux/vmlinux"
gdb -tui "$VMLINUX"

# then, inside gdb shell:
target remote :1234
{% endhighlight %}

To illustrate, below are two screenshots of this setup. It's from a different bug than
the one mentioned in this post, but it doesn't matter. I set a breakpoint at the *panic*
kernel function and then ran the reproducer:

![gdb](/~ricardo/assets/gdb_setup.png)
--- *Here you can see the virtual machine's console in the left, right before executing the
reproducer, and the host's gdb tui in the right.*

![gdb](/~ricardo/assets/gdb_panic.png)
--- *After executing the reproducer, the execution stops at the breakpoint we set
before, at the panic() function in the kernel/panic.c file.*

After working your way to tackle the problem, you can apply a patch to your local tree,
rebuild the kernel images and re-test with the reproducer until you get it right. Make
sure to see if the problem wasn't already fixed by someone else before sending the patch
upstream, though. Anyway, we didn't even scratch the surface here, but I hope this sheds
light into basic debugging techniques and make you better equipped to explore your way
into the kernel internals!


[dev-proc]: https://www.kernel.org/doc/html/latest/process/development-process.html
[mentorship]: https://mentorship.lfx.linuxfoundation.org/project/65d4c337-66fd-4d22-8a21-e836fafbebc4
[syzbot]: https://syzkaller.appspot.com/
[syzkaller]: https://github.com/google/syzkaller/
[syz-trees]: https://syzkaller.appspot.com/upstream/repos
[dmitry]: https://www.linuxfoundation.org/webinars/dynamic-program-analysis-for-fun-and-profit
[prado1]: https://sergioprado.blog/how-is-the-linux-kernel-tested/
[ldd4]: https://lwn.net/images/pdf/LDD3/ch04.pdf
[kernel-dbg]: https://www.kernel.org/doc/html/latest/dev-tools/kgdb.html
[tracing]: https://www.youtube.com/watch?v=JRyrhsx-L5Y
[prado2]: https://www.linuxfoundation.org/webinars/tools-and-techniques-to-debug-an-embedded-linux-system
[syz-docs]: https://github.com/google/syzkaller/blob/master/docs/linux/
[syz-assets]: https://github.com/google/syzkaller/blob/master/docs/syzbot_assets.md
[oops]: https://www.opensourceforu.com/2011/01/understanding-a-kernel-oops/
[initrd]: https://ibug.io/blog/2019/04/os-lab-1/
[kernel-init]: https://www.kernel.org/doc/html/latest/filesystems/ramfs-rootfs-initramfs.html
[buildroot]: https://buildroot.org/
[syz-img]: https://github.com/google/syzkaller/blob/master/tools/create-image.sh
[debootstrap]: https://wiki.debian.org/Debootstrap
[deb-releases]: https://wiki.debian.org/DebianReleases
[syz-bug]: https://groups.google.com/g/syzkaller-bugs/c/YIWmpQWCjT4/m/tw1tfm_xBAAJ
[syz-qemu]: https://github.com/google/syzkaller/blob/master/docs/syzbot.md#crash-does-not-reproduce
[qemu-doc]: https://qemu-project.gitlab.io/qemu/index.html
[gdb-docs]: https://www.sourceware.org/gdb/documentation/
