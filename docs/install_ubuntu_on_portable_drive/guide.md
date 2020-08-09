# 安装ubuntu到移动固态硬盘中，实现在多台电脑上都可启动的目标

# 需求
将ubuntu装入一个SSD中，实现随身携带的目的，并且可以随时插入另一台电脑使用SSD上的ubuntu系统。同理此文章不只适用ubuntu，大部分linux的安装思路是一样的。
# 设备
* surface pro 4, 5, 6（当然正常的支持ubuntu的电脑都是可以的）
* [Samsung T5 Portable SSD - 500GB - USB 3.1 External SSD (MU-PA500B/AM)](https://www.amazon.com/Samsung-T5-Portable-SSD-MU-PA500B/dp/B073GZBT36/ref=sr_1_3?ie=UTF8&qid=1535313863&sr=8-3&keywords=samsung+500gb+external+ssd)
* USB hub（surface pro只有一个USB接口）
* USB stick - 8GB（作为安装ubuntu的启动媒介）

# 步骤
## 1.下载ubuntu
这篇文章使用ubuntu桌面发行版[18.04.1 LTS](https://www.ubuntu.com/download/desktop)，下载对应的iso文件 - ubuntu-18.04.1-desktop-amd64.iso
## 2.将USB stick制作为启动盘
这里有多种方法：
* [Rufus](https://rufus.akeo.ie/)
* [Etcher](https://etcher.io/)
* linux的dd命令

## 3.安装系统
这里讲两种方法，使用ubuntu自带安装程序安装全新的系统，或者复刻已有的ubuntu系统
## 方法一：使用ubuntu自带安装程序安装系统

注意我们这里采用的是UEFI的方式，所以分区表采用GPT，最好给SSD新建一个GUID分区表。新建GPT可以用windows的工具，也可以用linux的命令行（参照方法二第一步）。

分区的话可以进入安装界面之后手动创建，分区及挂载点如下表：

| 分区类型                     | 大小   | 文件系统        | 挂载点    | 设备名称   |
|---                           |---     |---              |---        |---         |
| ESP（EFI System Partition）  | 300MB  | EFI filesystem  | /boot/efi | /dev/sdX1  |
| boot分区                     | 1GB    | ext4            | /boot     | /dev/sdX2  |
| 根分区                       | ~50GB  | ext4            |  /        | /dev/sdX3  |
| 与windows共享的分区 （可选） | ~400GB | ntfs            |  ~/share/ |  /dev/sdX4 |

这里的/dev/sdX指的是你的SSD设备。

安装一切都按正常步骤来就行，唯一需要注意的一点就是当已经分好分区并选择好各个分区的挂载点之后，一定要确保手动选择安装启动引导器的设备。
注意： “安装启动引导器的设备”一定要选SSD，即/dev/sdX，而不是SSD的某个分区，如/dev/sdX1

然而事实上，不知出于什么原因，ubuntu livecd 并没有将启动引导器安装到移动硬盘，开机会进入一个grub页面。(注: 貌似现在正常了, 不需要接下来的步骤了.)

如果要fix这个问题，可采取两种办法：

第一种：在grub界面通过ls命令来查看硬盘和分区，如果ls (hd0,3)/boot/grub显示信息，说明(hd0,3)是安装/boot的分区，可通过一下命令进入系统
```
set root=(hd0,3)
set prefix=(hd0,3)/boot/grub
insmod normal
normal
```

进入ubuntu系统后，输入
```shell
$ sudo grub-install /dev/sdX
$ sudo update-grub
```

第二种：使用USB stick烧录的livecd，不选择安装，选择进入试用ubuntu，然后按照第二种安装方法（跳过新建GPT和分区步骤即可）来操作即可


完成第一种或第二种之后，有可能需要挂载你原本系统的EFI分区，并手动删除ubuntu文件夹。


## 方法二：复刻已有的ubuntu系统
首先将SSD插入已有ubuntu系统的电脑
## 1.SSD的分区表（GPT）和分区的创建
注意我们这里采用的是UEFI的方式，所以分区表采用GPT
这里使用gdisk来操作SSD，输入
```shell
# gdisk /dev/sdX  sdX是SSD的名称
```
gdisk的使用很简单按照提示来就行
### 1.1 新建GUID Partition Table(GPT)
进入gdisk后，按下o就会新建一个GPT

### 1.2 创建分区

在gdisk中按下n会新建一个分区，起始和终止扇区（sector）决定了此分区的大小。
每次新建一个分区时，最后会要你设置partition type code，就是会有如下信息：
```
Current type is 'Linux filesystem'
Hex code or GUID (L to show codes, Enter = 8300):
```
系统默认是8300，也就是Linux filesystem。我们这里分三种，ESP这个分区要设置为EFI System，对应的type code是ef00，boot和根分区都是默认的Linux filesystem，type code是8300，与windows共享的分区用的是Microsoft basic data，其对应的type code是0700。

| 分区类型                     | 大小   | 文件系统          | 挂载点    | 设备名称     |
|---                         |---     |---              |---        |---         |
| ESP（EFI System Partition） | 300MB  | EFI filesystem  | /boot/efi | /dev/sdX1  |
| boot分区                    | 1GB    | ext4            | /boot     | /dev/sdX2  |
| 根分区                      | ~50GB  | ext4            |  /        | /dev/sdX3  |
| 与windows共享的分区 （可选）   | ~400GB | ntfs            |  ~/share/ |  /dev/sdX4 |

这里没有swap分区的原因有两点：
* 现在一般使用swap file代替swap分区， swap file是一个用户给定大小文件作为swap空间。相比于swap分区，它更灵活可以删除重新建立不同大小的swap空间
* 现在的电脑内存一般足够使用了，基本上用不到swap空间
想了解如何创建swap file可以自己google一下

格式化分区的命令：
```shell
# mkfs.vfat -F 32 /dev/sdX1
# mkfs.ext4 /dev/sdX2
# mkfs.ext4 /dev/sdX3
# mkfs.ntfs /dev/sdX4
```
## 2.挂载各个分区
创建分区并格式化后，如果SSD自动挂载，就先umount各个分区
```shell
# umount /dev/sdX1
# umount /dev/sdX2
# umount /dev/sdX3
# umount /dev/sdX4
``` 
接下来，挂载SSD的根分区到/mnt，并拷贝ubuntu的根目录到/mnt
```shell
# mount /dev/sdX3 /mnt
# cp -ax /* /mnt/
```
然后，挂载SSD的boot分区到/mnt/boot，并拷贝ubuntu的boot目录到/mnt/boot
```shell
# mount /dev/sdX2 /mnt/boot
# cp -ax /boot/* /mnt/boot/
```
最后，挂载SSD的EFI分区到/mnt/boot/efi，并拷贝ubuntu的efi目录到/mnt/boot/efi
```shell
# mount /dev/sdX1 /mnt/boot/efi
# cp -ax /boot/efi/* /mnt/boot/efi/
```
## 3.更改fstab文件中的分区UUID
使用命令blkid获取各个分区的UUID，共三个/dev/sdX1，/dev/sdX2 和 /dev/sdX3
```shell
# blkid
```
记录各个分区UUID，并替换/mnt/etc/fstab中对应的UUID值
```shell
# vim /mnt/etc/fstab
```

## 4.使用chroot将运行系统的根目录切换到SSD上的根目录分区
 
* /dev/sdX1 \-\-> EFI System Partition
* /dev/sdX2 \-\-> boot Partition
* /dev/sdX3 \-\-> root Partition

若未挂载，挂载各个分区和必要目录
```shell
# mount /dev/sdX3 /mnt
# mount /dev/sdX2 /mnt/boot
# mount /dev/sdX1 /mnt/boot/efi

# cd /mnt

# mount --bind /dev /mnt/dev
# mount --bind /dev/pts /mnt/dev/pts
# mount --bind /proc /mnt/proc
# mount --bind /sys /mnt/sys
# mount --bind /run /mnt/run
```

### 网络配置？（可不做）
```shell
# cp -L /etc/resolv.conf /mnt/etc/resolv.conf
```
改变根目录
```shell
# chroot /mnt
```

## 5.安装grub到SSD
```shell
# grub-install /dev/sdX
检查grub安装正确
# grub-install --recheck /dev/sdX
更新grub
# update-grub
退出chroot
# exit
# cd
卸载所有挂载
# umount --recursive /mnt
```
重启电脑并选择优先从SSD启动即可启动SSD上的ubuntu，同理此系统也可在其他机器上启动。

注意：在某些带有独立显卡的机器上，此SSD上的系统需安装相应的驱动方可启动


# 相关知识
## UEFI 和 BIOS
## GPT 和 MBR
## ESP
## Bootloader
## 5.将GRUB2安装到SSD的EFI System Partition
因为我们是用一台已经存在系统的电脑来安装ubuntu到移动SSD，一般来说ubuntu会将EFI bootloader安装到ESP上一个特殊地方：“fallback path” EFI\Boot\bootx64.efi 
