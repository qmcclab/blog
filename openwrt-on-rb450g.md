
> **NOTE** 本文严重参考http://wiki.hwmn.org/w/Mikrotik_RouterBoard_450G，不敢冒功，特此指出。

RB450G是MikkroTik公司出品的一款内置RouterOS的MIPS架构主板，CPU680Mhz，内存256M，nand512M，提供5个千兆网口，支持MicroSD和serial，性能远高于现市面上的无线路由设备。当然，它的售价也要高出许多。硬件规格高是一方面，另一个原因是RouterOS，它提供的路由、交换、VPN等功能特性非常丰富，目前在中低端市场，特别是欧洲占据了很大的市场份额。

RouterOS虽然基于linux，但却不开源，无法安装第三方软件包，其灵活性比OpenWRT差得多，因而本文就教大家如何把OpenWRT AA刷入RB450G。

准备：

- 1台PC，windows或linux均可；
- 1条null modern串口线；
- 1条普通网线；

windows需安装tftpd32和bitvise SSH server这两个软件，其中，tftpd32负责提供dhcp server功能，bitvise SSH server负责提供ssh服务。

nand说明

boot loader设置

# 启动

加电后，立刻在SecuCRT中敲任意键，进入`boot option`界面

成功后将通过dhcp服务器下载initram的OpenWRT内核，并顺利进入操作系统，目前操作系统运行在内存中，因而需要把真正的img刷入nand

内存区的一部分用于跑临时OpenWRT，剩余的125M挂载到`/tmp`目录。

# 备份

1. 通过winbox连接RB450G，备份出license.key，具体请参考RouterOS的官方文档。
2. 备份kernel和rootfs

        # cd /tmp
        # dd if=/dev/mtd5 | gzip -9 > routeros_kernel.img.gz
        # dd if=/dev/mtd6 | gzip -9 > routeros_rootfs.img.gz
        # scp routeros_kernel.img.gz routeros_rootfs.img.gz username@laptop.ip:/e//backup

> **WARN**
>
> 1. 创建rootfs.img.gz的过程非常慢，估计是gzip用了-9这个参数。
> 2. 创建rootfs.img.gz结束后，跳出xx的提示，
kernel.img.gz为1.8M，rootfs.img.gz为123.3M，尺寸之和两者正好是125.1M，因而不确定通过dd备份出来的rootfs是否完整。在原文中，作者也没有做倒换测试，后果自负。

# 下载

从`http://downloads.openwrt.org/attitude_adjustment/12.09/ar71xx/nand/`下载openwrt-ar71xx-nand-vmlinux.elf和openwrt-ar71xx-nand-rootfs.tar.gz这两个文件，并存放到笔记本的`e:\backup`目录中，一会儿会用到。

## 刷机

```bash
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

> **NOTE**
>
> `mtd erase rootfs`的时候，出现bad erase block的提示，我放狗搜了一下，发现是(正常现象)[http://wiki.openmoko.org/wiki/NAND_bad_blocks]，只要bad block的尺寸不超过1%即可放心使用。每个block大小为1/2KB。

刷完之后，reboot即可进入OpenWRT AA。

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

