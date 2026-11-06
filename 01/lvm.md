## LVM创建

#### 简介

> LVM（Logical Volume Manager，逻辑卷管理）在磁盘和文件系统之间加一层逻辑抽象，把物理设备组成存储池，再从池里划出可以动态扩容的逻辑卷，解决传统分区大小固定、扩容麻烦的问题。
> 分三层：PV 物理卷是把分区、裸盘或 md 阵列初始化成 LVM 可用的设备；VG 卷组由一个或多个 PV 组成，相当于存储池；LV 逻辑卷从卷组中划出来，格式化挂载后即可使用。本实验直接建在软RAID5阵列 /dev/md0 上，不预先分区。

创建RAID5（略，可参考软RAID.md），确认 md0 阵列已存在

```bash
[root@Server01 ~]# lsblk
NAME                   MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINTS
sr0                     11:0    1     8G  0 rom
nvme0n1                259:0    0   200G  0 disk
├─nvme0n1p1            259:1    0     1G  0 part
└─nvme0n1p2            259:2    0   199G  0 part
  ├─rhel_server01-root 253:0    0    70G  0 lvm   /
  ├─rhel_server01-swap 253:1    0   3.9G  0 lvm   [SWAP]
  └─rhel_server01-home 253:2    0 125.1G  0 lvm
nvme0n2                259:3    0   100G  0 disk
└─md0                    9:0    0 199.9G  0 raid5
nvme0n3                259:4    0   100G  0 disk
└─md0                    9:0    0 199.9G  0 raid5
nvme0n4                259:5    0   100G  0 disk
└─md0                    9:0    0 199.9G  0 raid5

# 注:md0 由三块 100G 盘组成 RAID5，容量 199.9G，LVM 直接建在这块阵列上
```

### 1.创建物理卷

```bash
[root@Server01 ~]# pvcreate /dev/md0
WARNING: ext4 signature detected on /dev/md0 at offset 1080. Wipe it? [y/n]: y
  Wiping ext4 signature on /dev/md0.
  Physical volume "/dev/md0" successfully created.
  Not creating system devices file due to existing VGs.
[root@Server01 ~]# pvs /dev/md0
  PV         VG  Fmt  Attr PSize   PFree
  /dev/md0   vg0 lvm2 a--  199.87g 199.87g

# 注:PV 物理卷设备名 VG 所属卷组 Fmt 元数据格式 Attr 物理卷属性 PSize 物理卷总容量 PFree 剩余可分配容量
# 注:设备上有旧的文件系统签名时会提示擦除，输入 y 确认
```

### 2.创建卷组

```bash
[root@Server01 ~]# vgcreate vg0 /dev/md0
  Not creating system devices file due to existing VGs.
  Volume group "vg0" successfully created
[root@Server01 ~]# vgs vg0
  VG  #PV #LV #SN Attr   VSize   VFree
  vg0   1   0   0 wz--n- 199.87g 199.87g

# 注:VG 卷组名 #PV 卷组内物理卷数量 #LV 逻辑卷数量 #SN 快照数量 Attr 卷组属性 VSize 卷组总容量 VFree 剩余可分配容量
```

### 3.创建逻辑卷

```bash
(1)创建逻辑卷
[root@Server01 ~]# lvcreate -L 100G -n lv0 vg0
  Logical volume "lv0" created.
[root@Server01 ~]# lvs /dev/vg0/lv0
  LV   VG  Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  lv0  vg0 -wi-a----- 100.05g

# 注:-L 100G 指定逻辑卷容量，-n lv0 指定逻辑卷名称，vg0 指定从哪个卷组划分
# 注:设备路径 /dev/vg0/lv0 和 /dev/mapper/vg0-lv0 是同一个设备

(2)查看三层状态
[root@Server01 ~]# pvscan
  PV /dev/nvme0n1p2   VG rhel_server01   lvm2 [<199.00 GiB / 0    free]
  PV /dev/md0         VG vg0             lvm2 [199.87 GiB / <99.87 GiB free]
  PV /dev/nvme0n5     VG vg0             lvm2 [<100.00 GiB / <100.00 GiB free]
  Total: 3 [498.86 GiB] / in use: 3 [498.86 GiB] / in no VG: 0 [0   ]

[root@Server01 ~]# vgscan
  Found volume group "rhel_server01" using metadata type lvm2
  Found volume group "vg0" using metadata type lvm2

[root@Server01 ~]# lvscan
  ACTIVE            '/dev/rhel_server01/swap' [<3.93 GiB] inherit
  ACTIVE            '/dev/rhel_server01/home' [125.07 GiB] inherit
  ACTIVE            '/dev/rhel_server01/root' [70.00 GiB] inherit
  ACTIVE            '/dev/vg0/lv0' [100.00 GiB] inherit
```

### 4.创建文件系统并挂载

```bash
[root@Server01 ~]# mkfs.ext4 /dev/vg0/lv0
mke2fs 1.46.5 (30-Dec-2021)
创建含有 26215424 个块（每块 4k）和 6561792 个inode的文件系统
文件系统UUID：e57146fe-9acd-4109-abb0-b3f19cd96e0e
超级块的备份存储于下列块：
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208,
        4096000, 7962624, 11239424, 20480000, 23887872

正在分配组表： 完成
正在写入inode表： 完成
创建日志（131072 个块）完成
写入超级块和文件系统账户统计信息： 已完成

[root@Server01 ~]# mount /dev/vg0/lv0 /newFS/
[root@Server01 ~]# cd /newFS/
[root@Server01 newFS]# echo "test" > test.txt
```

## LVM扩容

#### 简介

> 卷组空间不够时不用重做，先加一块盘做成 PV 扩到卷组里，再从卷组划给逻辑卷。扩容是三级接力，少做一步都白做：PV → VG → LV → 文件系统。

### 1.扩容卷组

```bash
(1)新加的硬盘先做成物理卷
[root@Server01 ~]# pvcreate /dev/nvme0n5
  Physical volume "/dev/nvme0n5" successfully created.
  Not creating system devices file due to existing VGs.

(2)加入卷组
[root@Server01 ~]# vgextend vg0 /dev/nvme0n5
  Volume group "vg0" successfully extended
[root@Server01 ~]# vgs vg0
  VG  #PV #LV #SN Attr   VSize    VFree
  vg0   2   1   0 wz--n- <299.87g <199.82g

# 注:VFree 变大只说明池子大了，逻辑卷容量没变，还要继续 lvextend
```

### 2.扩容逻辑卷

```bash
[root@Server01 ~]# lvextend -L +50G /dev/vg0/lv0
  Size of logical volume vg0/lv0 changed from 100.05 GiB (25613 extents) to 150.05 GiB (38413 extents).
  Logical volume vg0/lv0 successfully resized.
[root@Server01 ~]# lvs /dev/vg0/lv0
  LV   VG  Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  lv0  vg0 -wi-a----- 150.00g

# 注:lvextend 只扩了逻辑卷，还要扩文件系统才生效，xfs 用 xfs_growfs /挂载点，ext4 用 resize2fs /dev/vg0/lv0
```

## LVM缩容与删除

#### 简介

> 空间富余可以用 lvreduce 收回，整个卷不用了用 lvremove 删除。缩容要动文件系统，风险比扩容高得多；删除则必须先把 fstab 里对应的条目清掉。

### 1.缩小逻辑卷

```bash
[root@Server01 ~]# lvreduce -L -50G /dev/vg0/lv0
  WARNING: Reducing active logical volume to 100.00 GiB.
  THIS MAY DESTROY YOUR DATA (filesystem etc.)
Do you really want to reduce vg0/lv0? [y/n]: y
  Size of logical volume vg0/lv0 changed from 150.00 GiB (38401 extents) to 100.00 GiB (25601 extents).
  Logical volume vg0/lv0 successfully resized.
[root@Server01 ~]# lvs /dev/vg0/lv0
  LV   VG  Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  lv0  vg0 -wi-a----- 100.00g

# 注:本实验在格式化前缩容，没有数据风险；实际使用要先卸载并缩小文件系统再缩 LV，且只有 ext4 能缩，xfs 只能扩不能缩
```

### 2.删除逻辑卷、卷组、物理卷

```bash
[root@Server01 newFS]# cd
[root@Server01 ~]# umount /newFS
[root@Server01 ~]# lvremove /dev/vg0/lv0
Do you really want to remove active logical volume vg0/lv0? [y/n]: y
[root@Server01 ~]# vgremove /dev/vg0
[root@Server01 ~]# pvremove /dev/md0 /dev/nvme0n5

# 注:删除顺序从上往下，逻辑卷 → 卷组 → 物理卷
# 注:卷删了要顺手删掉 /etc/fstab 里对应的挂载条目，否则重启进 emergency mode
```
