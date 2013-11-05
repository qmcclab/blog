
> **NOTE** 本文严重参考[Mikro_RouterBoard_450G](http://wiki.hwmn.org/w/Mikrotik_RouterBoard_450G)，不敢冒功，特此指出。

RB450G是MikkroTik公司出品的一款内置RouterOS的MIPS架构主板，CPU680Mhz，内存256M，nand512M，提供5个千兆网口和serial口，支持MicroSD，性能远高于现市面上的无线路由设备。当然，它的售价也要高出许多。硬件规格高是一方面，另一个原因是RouterOS，它提供的路由、交换、VPN等功能特性非常丰富，目前在中低端市场，特别是欧洲占据了很大的市场份额。

RouterOS虽然基于linux，但却不开源，无法安装第三方软件包，其灵活性比OpenWRT差得多，因而本文就教大家如何把OpenWRT 12.09刷入RB450G。

## 工具

- 1台PC，提供dhcp和ssh服务，windows或linux均可；
- 1条null modern串口线+1个usb转RS232转接线；
- 1条网线；
- 下载OpenWRT 12.09的[vmlinux-initramfs.elf]()、[vmlinux.elf](http://downloads.openwrt.org/attitude_adjustment/12.09/ar71xx/nand/openwrt-ar71xx-nand-vmlinux.elf)和[rootfs.tar.gz](http:///downloads.openwrt.org/attitude_adjustment/12.09/ar71xx/nand/openwrt-ar71xx-nand-rootfs.tar.gz)

> **NOTE** 

我所使用的PC时windows，因而安装了tftpd32和bitvise SSH server这两个软件，其中，tftpd32负责提供dhcp server功能，bitvise SSH server负责提供ssh服务。

## RB450G介绍

> **NOTE** 以下说明适用于OpenWRT，不一定适用于RouterOS。

### nand和分区

nand和硬盘有些区别，它本身并无分区表的概念，直接由OpenWRT的kernel分配，因而在编译OpenWRT时就需要制定分区的大小，如下所示：

```
# dmesg | less
...
[    3.100000] Creating 3 MTD partitions on "NAND01GW3B2CN6":
[    3.110000] 0x000000000000-0x000000040000 : "booter"
[    3.110000] 0x000000040000-0x000000400000 : "kernel"
[    3.120000] 0x000000400000-0x000008000000 : "rootfs"
[    3.130000] mtd: partition "rootfs" set to be root filesystem
...
```

openwrt kernel将nand划分为三个分区：booter、kernel和rootfs。其中booter分区不可读，作用未知，剩余两个的作用就很明显了，kernel分区保存的openwrt kernel，rootfs则是openwrt 根文件系统，mountpoint也就是“/”。

**kernel分区**

kernel分区大小为4MiB，openwrt kernel尺寸在3.6-3.8MiB之间，加上yaffs的metadata和保留空间，正常情况下无法塞入kernel分区，因而openwrt耍了一个小聪明，将3.8MiB的kernel塞入6MiB[^footnote1]的kernel分区中。

### boot loader

RB450G提供了一个私有的bootloader：Routerboot。该bootloader提供了两份拷贝：一个保存于可读写的memory中，允许用户升级，甚至刷成第三方的bootloader；另一个保存于只读memory中，作为备份之用。当可读写的拷贝出现问题后，用户可以切换到备份拷贝，继续完成系统引导，所以说，RB450G的可玩性极强，不必担心刷成砖头。这是我喜欢RB450G的第二个原因。

### serial

RB450G提供了一个串口（serial），这对管理员来说非常方便，使用一台PC+一条null modern就可以登录操作系统进行调试，这是我喜欢RB450G的第三个原因。

在windows下，使用xshell或secucrt均可正常登录openwrt，linux下则建议使用kermit，minicom不靠谱。

### 交换机端口

```
  +-----------+       +-----------+
  |           | eth0  |           |
  |           +-------+----------5+-Eth1/PoE
  |           |       |           |
  |  AR7161   |       | AR8316 +-1+-Eth2
  |           | eth1  |        +-4+-Eth3
  |           +-------+0-------+-3+-Eth4
  |           |       |        +-2+-Eth5
  +-----------+       +-----------+
```

配置方式跟其它MIPS架构的交换机一样。

## 准备工作

**连接线缆**

将usb2rs232+null modern线将笔记本与RB450G的serial口相连。打开笔记本的“控制面板”，查看com口信息，然后打开SecuCRT创建一个serial连接，xx选择刚才看到的com口。

接着用网线连接笔记本和RB450G的eth0口，因为只有eth0口才是处于活动状态。

**tftpd32**

将`vmlinux-initramfs.elf`放在tftpd32的根目录，然后设置：



**bitvise SSH server**

没什么特别的，参照官方文档一步步设置就好了。主要用于openwrt的initram启动之后，备份RouterOS的kernel和rootfs。


## 刷机

**修改启动参数**

加电后，立刻在SecuCRT中敲任意键，进入`boot option`界面

成功后将通过dhcp server下载vmlinux-initramfs到内存，并顺利进入操作系统，目前操作系统运行在内存中，因而需要把真正的img刷入nand

内存区的一部分用于跑临时OpenWRT，剩余的125M挂载到`/tmp`目录。

**备份RouterOS**

1. 通过winbox连接RB450G，备份出license.key，具体请参考RouterOS的官方文档。
2. 备份RouterOS的kernel和rootfs

        # cd /tmp
        # dd if=/dev/mtd5 | gzip -9 > routeros_kernel.img.gz
        # dd if=/dev/mtd6 | gzip -9 > routeros_rootfs.img.gz
        # scp routeros_kernel.img.gz routeros_rootfs.img.gz username@laptop.ip:/e//backup

> **NOTE**
>
> 1. 创建rootfs.img.gz的过程非常慢，估计是gzip用了-9这个参数。
> 2. 创建rootfs.img.gz结束后，跳出xx的提示，
kernel.img.gz为1.8M，rootfs.img.gz为123.3M，尺寸之和两者正好是125.1M，因而不确定通过dd备份出来的rootfs是否完整。在原文中，作者也没有做倒换测试，后果自负。

**灌入**

```
# cd /tmp
# scp username@laptop.ip:/e//backup//openwrt-ar71xx-nand-vmlinux.elf .
# scp username@laptop.ip:/e//backup//openwrt-ar71xx-nand-rootfs.tar.gz .
# mtd erase kernel
# mount -t yaffs /dev/mtdblock5 /mnt
# cp openwrt-ar71xx-nand-vmlinux.elf /mnt/kernel
# umount /mnt
# mtd erase rootfs
# mount -t yaffs /dev/mtdblock6 /mnt
# cd /mnt
# tar xpzf /tmp/openwrt-ar71xx-nand-rootfs.tar.gz
# cd /tmp
# umount /mnt
```

**NOTE** `mtd erase rootfs`的时候，出现bad erase block的提示，这是[正常现象](http://wiki.openmoko.org/wiki/NAND_bad_blocks)，只要bad block的尺寸不超过nand大小的1%即可放心使用。每个block大小为1/2KB。

刷完之后，reboot即可进入OpenWRT 12.09。

## nand bad block

http://wiki.openmoko.org/wiki/NAND_bad_blocks有详细的说明，以下是网友的回复。

This is only a warning.

It is issued by the MTD driver while scanning for bad blocks in the flash
device. see: drivers/mtd/nand/nand_bbt.c :

printk (KERN_WARNING "Bad eraseblock %d at 0x%08x\n", i >> 1, (unsigned int)
from);


The device is still good!

NAND devices, like hard drives, ship with bad blocks to increase yield, and
reducing cost. They also develop bad blocks over time. The Kernel keeps track
of these bad blocks so the filesystem knows not to use them.

Each block consists of 32 pages, and each page has a 1/2 K. So you lose 16 KB
for each bad block. (Which isn't bad for 128MB flash system).

Unfortunately the TS boot code does not check for errors or bad blocks when it
loads the 256KB Redboot image from flash. So if you have a bad block in this
area, your system could fail to boot with no error message.

I have build a 2K ARM assembly language boot loader that will correct and
detect errors as it loads the 256KB Redboot image from flash. It is part of
the "Serial Blaster" project (to be launched in alpha version, for the
TS7250, soon).

-Curtis.

源文档 <http://tech.groups.yahoo.com/group/ts-7000/message/1368>

The TS-7250 uses Nand flash, Nand flash is much different than Nor
flash(if you are interested there is a pretty good write up here
http://support.gateway.com/s/Manuals/Desktops/5502664/NOR_vs_NANDwhitepaper.pdf)\
.
Basically Nand flash is sometimes shipped from the manufacturer with bad
blocks, in addition blocks will become bad over time. The general
consensus is not to worry if your flash consists of 1% or less bad
blocks. Nand flash has out of band data which basically stores meta data,
this is where a block is marked bad. The filesystem(Yaffs) handles this
and will ensure data is not written to any block marked as bad. The
bottom line is the messages your are seeing are benign, unless the number
of bad blocks exceeds 1%.

源文档 <http://tech.groups.yahoo.com/group/ts-7000/message/1367>

