---
layout: post
title: "LFX Linux Kernel Bug Fixing Fall"
tags: linux kernel mentorship
---

intro lfx mentorship

intro to kernel debugging and resources (+gdb)

introduce KASAN et al, syzkaller, mention [dmitry]

basic workflow
-	choose bug
-	reproduce
-	fix*
-	send patch
*
-	checkout commit
-	copy config
-	patch tree
-	build
-	boot kernel with trixie.img
-	reproduce?

{% highlight bash %}
# https://github.com/google/syzkaller/blob/master/tools/create-image.sh
./create-image.sh --distribuition trixie
# see all options and customization
# i.e. currently it is missing wget

qemu-system-x86_64 \
	-m 2G \
	-smp 2,sockets=2,cores=1 \
	-net nic,model=e1000 \
	-net user,host=10.0.2.25,hostfwd=tcp::10022-:22 \
	-enable-kvm \
	-nographic \
	-machine pc-q35-7.1 \
	-append "root=/dev/sda console=ttyS0 earlyprintk=serial net.ifnames=0" \
	-drive format=raw,file=/home/rbmarliere/images/syzkaller/trixie.img \
	-snapshot \
	-kernel

# https://github.com/google/syzkaller/blob/master/docs/syzbot_assets.md#run-a-c-reproducer
qemu-system-x86_64 \
	-m 2G \
	-smp 2,sockets=2,cores=1 \
	-drive file=./disk\
	-40f71e7c.raw,format=raw \
	-net nic,model=e1000 \
	-net user,host=10.0.2.10,hostfwd=tcp::10022-:22 \
	-enable-kvm \
	-nographic \
	-snapshot \
	-machine pc-q35-7.1

# https://github.com/google/syzkaller/blob/master/docs/linux/setup_ubuntu-host_qemu-vm_x86-64-kernel.md#verify
qemu-system-x86_64 \
	-m 2G \
	-smp 2 \
	-kernel $KERNEL/arch/x86/boot/bzImage \
	-append "console=ttyS0 root=/dev/sda earlyprintk=serial net.ifnames=0" \
	-drive file=$IMAGE/bullseye.img,format=raw \
	-net user,host=10.0.2.10,hostfwd=tcp:127.0.0.1:10021-:22 \
	-net nic,model=e1000 \
	-enable-kvm \
	-nographic \
	-pidfile vm.pid \
	2>&1 | tee vm.log

# https://github.com/google/syzkaller/blob/master/docs/syzbot.md#crash-does-not-reproduce
qemu-system-x86_64 \
	-smp 2 \
	-m 4G \
	-enable-kvm \
	-cpu host \
	-net nic \
	-net user,hostfwd=tcp::10022-:22 \
	-kernel arch/x86/boot/bzImage -nographic \
	-device virtio-scsi-pci,id=scsi \
	-device scsi-hd,bus=scsi.0,drive=d0 \
	-drive file=stretch-amd64.img,format=raw,if=none,id=d0 \
	-append "root=/dev/sda1 console=ttyS0 earlyprintk=serial"

# https://github.com/hardenedlinux/Debian-GNU-Linux-Profiles/blob/master/docs/harbian_qa/fuzz_testing/syz_debug.md#check-vm
qemu-system-x86_64 \
	-m 2048 \
	-net nic \
	-net user,host=10.0.2.10,hostfwd=tcp::15965-:22 \
	-display none \
	-no-reboot \
	-enable-kvm \
	-hda /root/linux_img/syzkalls.img \
	-snapshot \
	-initrd initrd.img \
	-kernel bzImage \
	-append console=ttyS0 vsyscall=native rodata=n root=/dev/sda1

# https://qemu-project.gitlab.io/qemu/system/linuxboot.html
qemu-system-x86_64 \
	-kernel bzImage \
	-hda rootdisk.img \
	-append "root=/dev/hda console=ttyS0" \
	-nographic

# https://ibug.io/blog/2019/04/os-lab-1/
qemu-system-i386 \
	-kernel linux-2.6.26/arch/x86/boot/bzImage \
	-initrd myinitrd.img \
	--append "root=/dev/ram init=/init"


alias ssh-='ssh -i /home/rbmarliere/images/syzkaller/trixie.id_rsa -p 10022 root@localhost'
alias scp-='scp -i /home/rbmarliere/images/syzkaller/trixie.id_rsa -P 10022 '
{% endhighlight %}


[syzkaller-docs]: https://github.com/google/syzkaller/blob/master/docs/linux/
[dmitry]: https://www.youtube.com/watch?v=ufcyOkgFZ2Q
