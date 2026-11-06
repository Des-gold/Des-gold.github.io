## 软RAID5部署

#### 简介

> RAID（Redundant Array of Independent Disks，独立磁盘冗余阵列）把多块物理磁盘组合成一个逻辑设备使用，用多块盘换性能、换容量或换安全。常见级别有 RAID0（纯条带，无冗余）、RAID1（镜像）、RAID5（分布式校验，牺牲一块盘容量换容错）、RAID6（双校验，容两块盘故障）、RAID10（先镜像再条带）。
> RAID 有硬 RAID 和软 RAID 两种实现，硬 RAID 靠独立阵列卡，软 RAID 由内核 md 驱动完成，用 mdadm 管理，不需要额外硬件。本实验做的是软 RAID5。

### 1.查看硬盘

添加 4 块 NVME 100G 硬盘（略），设备名为 nvme0n[2-5]

```bash
[root@Server01 ~]# lsblk
NAME                   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sr0                     11:0    1     8G  0 rom
nvme0n1                259:0    0   200G  0 disk
├─nvme0n1p1            259:1    0     1G  0 part
└─nvme0n1p2            259:2    0   199G  0 part
  ├─rhel_server01-root 253:0    0    70G  0 lvm  /
  ├─rhel_server01-swap 253:1    0   3.9G  0 lvm  [SWAP]
  └─rhel_server01-home 253:2    0 125.1G  0 lvm
nvme0n2                259:3    0   100G  0 disk
nvme0n3                259:7    0   100G  0 disk
nvme0n4                259:8    0   100G  0 disk
nvme0n5                259:9    0   100G  0 disk
```

### 2.创建RAID5

3 块盘做阵列成员，1 块做热备，可用容量为 (3-1)×100G = 200G

```bash
(1)安装 mdadm
[root@Server01 ~]# dnf install mdadm -y
...
完毕！

(2)创建阵列
[root@Server01 ~]# mdadm --create /dev/md0 --level=5 --raid-devices=3 /dev/nvme0n[2-4] --spare-devices=1 /dev/nvme0n5
To optimize recovery speed, it is recommended to enable write-intent bitmap, do you want to enable it now? [y/N]? y
...
Continue creating array [y/N]? y
mdadm: Defaulting to version 1.2 metadata
mdadm: array /dev/md0 started.

# 注:--create 创建阵列，/dev/md0 阵列设备名，--level=5 级别 RAID5，--raid-devices=3 成员数量，--spare-devices=1 热备数量
```

### 3.查看阵列信息

```bash
[root@Server01 ~]# mdadm --detail /dev/md0
/dev/md0:
           Version : 1.2
     Creation Time : Thu Nov  5 22:16:56 2026
        Raid Level : raid5
        Array Size : 209582080 (199.87 GiB 214.61 GB)
     Used Dev Size : 104791040 (99.94 GiB 107.31 GB)
      Raid Devices : 3
     Total Devices : 4
...
    Active Devices : 2
   Working Devices : 4
    Failed Devices : 0
     Spare Devices : 2

...
    Number   Major   Minor   RaidDevice State
       0     259        3        0      active sync   /dev/nvme0n2
       1     259        7        1      active sync   /dev/nvme0n3
       4     259        8        2      active sync   /dev/nvme0n4
       3     259        9        -      spare         /dev/nvme0n5

# 注:active sync 正常成员，spare 热备盘，faulty 故障盘
```

### 4.创建文件系统并挂载

```bash
(1)格式化阵列
[root@Server01 ~]# mkfs.ext4 /dev/md0
mke2fs 1.46.5 (30-Dec-2021)
创建含有 78593280 个块（每块 4k）和 19652608 个inode的文件系统
文件系统UUID：112cf30d-7a67-4e26-8d77-0162ffd453cb
超级块的备份存储于下列块：
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208,
        4096000, 7962624, 11239424, 20480000, 23887872, 71663616

正在分配组表： 完成
正在写入inode表： 完成
创建日志（262144 个块）完成
写入超级块和文件系统账户统计信息： 已完成

(2)挂载并写入测试文件
[root@Server01 ~]# mkdir /newFS
[root@Server01 ~]# mount /dev/md0 /newFS/
[root@Server01 ~]# ln -s /newFS/ newFS

[root@Server01 newFS]# echo "test" > test.txt
```

## 阵列故障处理

#### 简介

> RAID5 允许坏一块盘。这里手动标记一块成员盘失效并移除，观察热备盘自动接替失效盘位置、阵列开始重构的过程。

### 1.模拟磁盘故障

```bash
(1)标记并移除一块成员盘
[root@Server01 ~]# mdadm /dev/md0 --fail /dev/nvme0n2
[root@Server01 ~]# mdadm /dev/md0 --remove /dev/nvme0n2
mdadm: hot removed /dev/nvme0n2 from /dev/md0

# 注:先 --fail 再 --remove，只 fail 不 remove 的话坏盘一直占着槽位，新盘加不进去

(2)查看重构状态
[root@Server01 ~]# mdadm --detail /dev/md0
/dev/md0:
           Version : 1.2
     Creation Time : Thu Nov  5 22:16:56 2026
        Raid Level : raid5
        Array Size : 209582080 (199.87 GiB 214.61 GB)
     Used Dev Size : 104791040 (99.94 GiB 107.31 GB)
      Raid Devices : 3
     Total Devices : 3
...
    Active Devices : 2
   Working Devices : 3
    Failed Devices : 0
     Spare Devices : 1
...
    Rebuild Status : 37% complete

              Name : Server01:0  (local to host Server01)
              UUID : d9ddf650:aa134655:baaca817:6d0ffa47
            Events : 151

    Number   Major   Minor   RaidDevice State
       3     259        9        0      spare rebuilding   /dev/nvme0n5
       1     259        7        1      active sync   /dev/nvme0n3
       4     259        8        2      active sync   /dev/nvme0n4

# 注:热备盘已接替失效盘的位置，Rebuild Status 走到 100% 前不要重启，此时阵列零冗余
```

### 2.停止并清理阵列

```bash
[root@Server01 ~]# umount /dev/md0
[root@Server01 ~]# mdadm -S /dev/md0
mdadm: stopped /dev/md0
[root@Server01 ~]# mdadm --misc --zero-superblock /dev/nvme0n[2-5]

# 注:zero-superblock 不清的话，下次开机内核会自动把阵列重新组装起来
```
