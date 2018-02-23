---
title: Ubuntu分区删除后恢复
date: 2016/08/13 19:22:11
toc: false
list_number: false
categories:
- Ubuntu
tags:
- Ubuntu
---

## 前言
由于一次手误，在Windows中把Ubuntu的系统分区和swap分区删除了，导致不能进系统。开机直接`error no such partition`和`grub rescue mode`。下面讲述一下系统恢复过程！
> 系统是Windows7和Ubuntu14.04双系统。DELL，256SSD，8GMEM；

## 一，恢复grub
- 现在已经无法进入系统，按照网上的方法重新设置grub引导已经无效（因为引导分区已经删除，ls是看不到引导分区的），所以，进入pe，选择恢复删除的两个分区，重启。
- 如果能正常进入系统，恭喜，ok了。如果不行，继续进入pe，选择修复grub2分区，重启。
- ok还是进入grub rescue mode，这时设置引导分区：
```
1. ls查找磁盘分区：（hd0） （hd0，msdos1）（hd0，msdos2）（hd0，msdos3）（hd0，msdos4）
2. ls (hd0,msdosx)/boot/grub：直到找到grub所在分区，也就是你Ubuntu系统装的分区。
3. set root=(hd0,msdos3)
3. set prefix=(hd0,msdos3)/boot/grub
4. insmod normal
5. normal回车
```
- 如果成功进入系统那么恭喜你！如果提示`grub_term_highlight`那么继续吧。
- 使用usb制作的Ubuntu启动盘进入Try Ubuntu模式，打开terminal输入命令：
```
1. sudo fdisk -l #查找Ubuntu按照分区
2. sudo mount /dev/sda3 /mnt #挂载分区
3. sudo grub-install --root-directory=/mnt/dev/sda #安装
4.重启，成功进入系统
5. 更新grub：sudo update-grub2
```
至此，Ubuntu系统又回来了！但是不要忘了，我们把swap分区也删了，重新恢复分区，会导致分区UUID改变，所以swap分区无法使用！
## 二，挂载swap分区
使用`sudo fdisk -l`可以看见swap分区还存在，但是`free -m`却发现swap分区都是0。那么继续吧！
- `sudo mkswap -L swap_sda6 /dev/sda6` 格式化swap分区（我的swap分区是在/dev/sda6），并记录下UUID。
```
Setting up swapspace version 1, size = 8290292 KiB
LABEL=swap_sda6, UUID=df92467e-3548-45e6-bc30-2f37c6eda614
```
-  修改/etc/fstab配置文件，实现挂载
将`UUID=ee3b8097-7c2e-47d0-8188-d6d6ra342cb3 none  swap    sw    defaults      0  0`这一行中的UUID换成刚刚记录的UUID，保存退出。
- `swapon -a`激活，然后样`free -m`或`free -h`查看！也可以使用`swapon -s`查看！
完成！由于Google没有找到解决方案，故记录之。另外，想想要修改swap分区其实也是很简单的事情了！