## 磁盘分区管理配置

#### 简介

> fdisk 是 Linux 下最常用的分区表操作工具，支持 MBR（dos）和 GPT 两种分区表，默认 dos。它只负责修改分区表，不做格式化，改完必须输入 w 写入磁盘才生效，只按 q 退出等于什么都没做。
> MBR 分区表最多支持 4 个主分区，或者 3 个主分区 + 1 个扩展分区。扩展分区本身不存数据，只是逻辑分区的容器，逻辑分区编号从 5 开始。
> 交互命令：p 打印分区表、n 新建分区、d 删除分区、t 修改分区类型、w 保存退出、q 不保存退出。

| 硬件设备 | 文件名称 |
| --- | --- |
| IDE 设备 | /dev/hd[a-d] |
| SCSI/SATA/U盘 | /dev/sd[a-p] |
| 非易失性存储器标准(Non-Volatile Memory Express, NVMe)硬盘 | /dev/nvme0n[1-m]，例如 /dev/nvme0n1 就是第一个 NVMe 硬盘 |

### 1.查看硬盘

添加一块 NVME 100G 硬盘(略)，实验环境以 VMware 虚拟机为例，关机状态下添加，开机后系统自动识别为 /dev/nvme0n2

```bash
[root@Server01 ~]# lsblk
NAME                   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sr0                     11:0    1     8G  0 rom
nvme0n1                259:0    0   200G  0 disk
├─nvme0n1p1            259:1    0     1G  0 part /boot
└─nvme0n1p2            259:2    0   199G  0 part
  ├─rhel_server01-root 253:0    0    70G  0 lvm  /
  ├─rhel_server01-swap 253:1    0   3.9G  0 lvm  [SWAP]
  └─rhel_server01-home 253:2    0  125.1G 0 lvm  /home
nvme0n2                259:3    0   100G  0 disk

# 注:nvme0n2 就是我们新添加的 100G 硬盘，TYPE 为 disk 且下面没有 part，说明还没有分区
```

### 2.创建主分区

```bash
(1)进入 fdisk 创建分区
[root@Server01 ~]# fdisk /dev/nvme0n2
...
命令(输入 m 获取帮助)：p
Disk /dev/nvme0n2：100 GiB，107374182400 字节，209715200 个扇区
磁盘型号：VMware Virtual NVMe Disk
磁盘标签类型：dos
磁盘标识符：0xfb743eb5

命令(输入 m 获取帮助)：n
分区类型
   p   主分区 (0 primary, 0 extended, 4 free)
   e   扩展分区 (逻辑分区容器)
选择 (默认 p)：p
分区号 (1-4, 默认  1): 1
第一个扇区 (2048-209715199, 默认 2048):
最后一个扇区，+/-sectors 或 +size{K,M,G,T,P} (2048-209715199, 默认 209715199): +20G

创建了一个新分区 1，类型为"Linux"，大小为 20 GiB。

命令(输入 m 获取帮助)：w
分区表已调整。
将调用 ioctl() 来重新读分区表。
正在同步磁盘。

(2)查看分区结果
[root@Server01 ~]# lsblk
NAME                   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sr0                     11:0    1     8G  0 rom
nvme0n1                259:0    0   200G  0 disk
├─nvme0n1p1            259:1    0     1G  0 part /boot
└─nvme0n1p2            259:2    0   199G  0 part
  ├─rhel_server01-root 253:0    0    70G  0 lvm  /
  ├─rhel_server01-swap 253:1    0   3.9G  0 lvm  [SWAP]
  └─rhel_server01-home 253:2    0  125.1G 0 lvm  /home
nvme0n2                259:3    0   100G  0 disk
└─nvme0n2p1            259:5    0    20G  0 part

# 注:nvme0n2 硬盘下新增一个分区 nvme0n2p1 为 20G
```

### 3.创建逻辑分区

创建逻辑分区之前必须先创建扩展分区，因为 MBR 分区表中逻辑分区只能存在于扩展分区内部

```bash
(1)创建扩展分区，再在其中创建逻辑分区
[root@Server01 ~]# fdisk /dev/nvme0n2
...
命令(输入 m 获取帮助)：n
分区类型
   p   主分区 (1 primary, 0 extended, 3 free)
   e   扩展分区 (逻辑分区容器)
选择 (默认 p)：e
分区号 (2-4, 默认  2):
第一个扇区 (41945088-209715199, 默认 41945088):
最后一个扇区，+/-sectors 或 +size{K,M,G,T,P} (41945088-209715199, 默认 209715199):

创建了一个新分区 2，类型为"Extended"，大小为 80 GiB。

命令(输入 m 获取帮助)：n
所有主分区的空间都在使用中。
添加逻辑分区 5
第一个扇区 (41947136-209715199, 默认 41947136):
最后一个扇区，+/-sectors 或 +size{K,M,G,T,P} (41947136-209715199, 默认 209715199): +10G

创建了一个新分区 5，类型为"Linux"，大小为 10 GiB。

命令(输入 m 获取帮助)：p
Disk /dev/nvme0n2：100 GiB，107374182400 字节，209715200 个扇区
磁盘型号：VMware Virtual NVMe Disk
磁盘标签类型：dos
磁盘标识符：0xfb743eb5

设备           启动     起点      末尾      扇区 大小 Id 类型
/dev/nvme0n2p1          2048  41945087  41943040  20G  0 空
/dev/nvme0n2p2      41945088 209715199 167770112  80G  5 扩展
/dev/nvme0n2p5      41947136  62918655  20971520  10G 83 Linux

命令(输入 m 获取帮助)：w
分区表已调整。
将调用 ioctl() 来重新读分区表。
正在同步磁盘。

(2)查看分区结果
[root@Server01 ~]# lsblk
NAME                   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sr0                     11:0    1     8G  0 rom
nvme0n1                259:0    0   200G  0 disk
├─nvme0n1p1            259:1    0     1G  0 part /boot
└─nvme0n1p2            259:2    0   199G  0 part
  ├─rhel_server01-root 253:0    0    70G  0 lvm  /
  ├─rhel_server01-swap 253:1    0   3.9G  0 lvm  [SWAP]
  └─rhel_server01-home 253:2    0  125.1G 0 lvm  /home
nvme0n2                259:3    0   100G  0 disk
├─nvme0n2p1            259:8    0    20G  0 part
├─nvme0n2p2            259:9    0     1K  0 part
└─nvme0n2p5            259:10   0    10G  0 part

# 注:nvme0n2p2 显示 1K 是正常的，扩展分区只是容器不存数据，真正可用的是里面的逻辑分区 nvme0n2p5
```

## 文件系统与挂载

#### 简介

> 分区只是划出了边界，里面还是空的，必须格式化创建文件系统才能存数据。挂载是把文件系统接到目录树上，mount 是临时挂载、重启失效，写进 /etc/fstab 才能开机自动挂载。

### 1.创建文件系统并挂载

```bash
[root@Server01 ~]# mkfs.ext4 /dev/nvme0n2p1
...
正在分配组表： 完成
正在写入inode表： 完成
创建日志（32768 个块）完成
写入超级块和文件系统账户统计信息： 已完成

[root@Server01 ~]# mkdir /newFS
[root@Server01 ~]# mount /dev/nvme0n2p1 /newFS/
[root@Server01 ~]# df -h
文件系统                        容量  已用  可用 已用% 挂载点
devtmpfs                        1.8G     0  1.8G    0% /dev
tmpfs                           1.9G     0  1.9G    0% /dev/shm
tmpfs                           740M  9.1M  731M    2% /run
/dev/mapper/rhel_server01-root   70G  1.8G   69G    3% /
/dev/nvme0n1p1                 1014M  197M  818M   20% /boot
/dev/mapper/rhel_server01-home  126G  925M  125G    1% /home
tmpfs                           370M     0  370M    0% /run/user/0
/dev/nvme0n2p1                   20G   24K   19G    1% /newFS

# 注:mount 挂载是临时的，重启后失效
```

### 2.卸载

```bash
[root@Server01 ~]# umount /dev/nvme0n2p1
[root@Server01 ~]# df -h
文件系统                        容量  已用  可用 已用% 挂载点
devtmpfs                        1.8G     0  1.8G    0% /dev
tmpfs                           1.9G     0  1.9G    0% /dev/shm
tmpfs                           740M  9.1M  731M    2% /run
/dev/mapper/rhel_server01-root   70G  1.8G   69G    3% /
/dev/nvme0n1p1                 1014M  197M  818M   20% /boot
/dev/mapper/rhel_server01-home  126G  925M  125G    1% /home
tmpfs                           370M     0  370M    0% /run/user/0

# 注:提示 target is busy 说明有进程正占用挂载点，用 lsof /newFS 找到进程退出后再卸
```

### 3.自动挂载

通过 /etc/fstab 实现开机自动挂载，用 UUID 而不是设备名，防止设备名变化导致挂载失败

```bash
(1)查询 UUID 并写入 fstab
[root@Server01 ~]# blkid | grep nvme0n2p1
/dev/nvme0n2p1: UUID="2a93a191-b571-4a4a-9f08-bb03e1f59c36" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="fb743eb5-01"

[root@Server01 ~]# vi /etc/fstab
# 添加
UUID=2a93a191-b571-4a4a-9f08-bb03e1f59c36 /newFS ext4 defaults 0 0

[root@Server01 ~]# cat /etc/fstab
...
/dev/mapper/rhel_server01-root /                       xfs     defaults        0 0
UUID=3fde47cf-b570-40bc-9bf0-4439b0cbbe1f /boot       xfs     defaults        0 0
/dev/mapper/rhel_server01-home /home                   xfs     defaults        0 0
/dev/mapper/rhel_server01-swap none                    swap    defaults        0 0
UUID=2a93a191-b571-4a4a-9f08-bb03e1f59c36 /newFS      ext4    defaults        0 0

# 注:每块盘的 UUID 随机生成都不一样，要换成自己 blkid 查到的值

(2)验证 fstab 是否写对
[root@Server01 ~]# umount /newFS
[root@Server01 ~]# mount -a
[root@Server01 ~]# df -h | grep new
/dev/nvme0n2p1                   20G   24K   19G    1% /newFS

# 注:mount -a 会挂载 fstab 里所有未挂载的设备，能挂上说明写对了；fstab 写错会导致系统开不了机，改完务必先验证再重启
```
