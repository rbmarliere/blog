---
layout: post
title: "Booting the Raspberry Pi 3 from the network"
date: 2023-10-23 17:00:00 -0300
tags: linux kernel mentorship
---

The use case is simple: boot a kernel from [tftp][tftp], mount a root filesystem through
[NFS][nfs]. From the minimal kernel built [before][rpi], you need those symbols:

```
INPUT_KEYBOARD       Enables keyboard drivers selection submenu
INET                 TCP/IP networking
NFS_FS               Network FS client support (v3)
IP_PNP               Automatic IP configuration
IP_PNP_DHCP          DHCP for IP auto config
ROOT_NFS             Root FS through NFS

NETDEVICES           Network device support
USB_USBNET           USB Networking
USB_NET_SMSC95XX     SMSC LAN95XX Ethernet adapter driver
```

You could drop MMC, since you only need a single FAT32 partition containing the
[bootloader][u-boot] and firmware:

```
start.elf
bootcode.bin
fixup.dat
config.txt
u-boot.bin
boot.scr
```

The first three are the same firmware files from before and **config.txt** will have the
following content:

```
arm_64bit=1
kernel=u-boot.bin
```

After [building][u-boot-git] **u-boot.bin**, go ahead and create a **boot.cmd** script
file like so:

```
setenv serverip 192.168.1.100
setenv ipaddr 192.168.1.101
setenv bootargs root=/dev/nfs rw nfsroot=192.168.1.100:/exports/aarch64,nfsvers=3 ip=dhcp
tftpboot ${fdt_addr_r} dtb
tftpboot ${kernel_addr_r} kernel
booti ${kernel_addr_r} - ${fdt_addr_r}
```

Build it with:

```
$GIT_DIR/u-boot/tools/mkimage -A arm64 -O linux -T script -C none -d boot.cmd boot.scr
```

This script assumes that your host 192.168.1.100 serves both [tftp][tftp] and
[nfs-kernel-server][nfs-utils]. You need something like this in your **/etc/exports** file:

```
/exports/aarch64 *(rw,sync,no_root_squash,no_subtree_check)
```

It also expects both **dtb** and **kernel** files to be present in the tftp share root.
Instead of pin-pointing to version names, you can use symbolic links, e.g.:

```
cd /srv/tftp
ln -s vmlinuz-6.6.0-rc1+ kernel
ln -s broadcom/bcm2837-rpi-3-b.dtb dtb
```

The links can't point to outside the share root, though.

![nfs](/~ricardo/assets/rpi_nfs.png)
--- *Profit.*

&nbsp;

---

&nbsp;

#### References
- <https://elinux.org/RPi_U-Boot>
- <https://sergioprado.org/utilizando-o-u-boot-na-raspberry-pi/>
- <https://github.com/mhomran/u-boot-rpi3-b-plus>


[rpi]: {% link _posts/2023-10-12-raspberrypi.markdown %}
[tftp]: https://tracker.debian.org/pkg/tftpd-hpa
[nfs]: https://www.kernel.org/doc/Documentation/filesystems/nfs/nfsroot.txt
[u-boot]: https://u-boot.readthedocs.io/en/latest/usage/index.html#shell-commands
[u-boot-git]: https://source.denx.de/u-boot/u-boot
[nfs-utils]: https://tracker.debian.org/pkg/nfs-utils
