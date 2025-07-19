## IPSAN服务部署

#### 简介

> IPSAN的核心技术是iSCSI（Internet SCSI），即Internet SCSI，是IETF制订的一项标准。iSCSI技术将存储行业广泛应用的SCSI接口技术与IP网络技术相结合，可以在IP网络上构建SAN。简单地说，iSCSI就是在IP网络上运行SCSI协议的一种网络存储技术

### 服务端

#### 环境

| 设备             | 网卡               | GW            |
| ---------------- | ------------------ | ------------- |
| Server01(服务端) | 192.168.100.131/24 | 192.168.100.2 |
| Server02(客户端) | 200.168.100.132/24 | 192.168.100.2 |

注:Server01(服务端)添加一块硬盘进行实验,下面我以nvme0n2(20G)为例子

#### 安装ISCSI服务

```
[root@server01 ~]# dnf install targetcli –y
[root@server01 ~]# systemctl enable --now target
```

#### 创建共享设备

```
[root@server01 ~]# targetcli 
targetcli shell version 2.1.fb49
Copyright 2011-2013 by Datera, Inc and others.
For help on commands, type 'help'.
 
/> ls 
o- / ....................................................................................... [...]
  o- backstores ............................................................................ [...]
  | o- block ................................................................ [Storage Objects: 0]
  | o- fileio ............................................................... [Storage Objects: 0]
  | o- pscsi ................................................................ [Storage Objects: 0]
  | o- ramdisk .............................................................. [Storage Objects: 0]
  o- iscsi .......................................................................... [Targets: 0]
  o- loopback ....................................................................... [Targets: 0]
 
====添加ISCSI后端存储====
/> cd /backstores/block 
/backstores/block> create data  /dev/nvme0n2
 
====设置IQN共享名称====
/backstores/block> cd /iscsi 
/iscsi> create iqn.2024-07.com.shaer-disk:server 
 
====设置访问IQN共享的设备名称====
/backstores/block> cd /iscsi 
/iscsi> create iqn.2024-07.com.shaer-disk:server 
/iscsi> cd iqn.2024-07.com.shaer-disk:server/tpg1/acls/
/iscsi/iqn.20...ver/tpg1/acls> create  iqn.2024-07.com.shaer-disk:client01
/iscsi/iqn.20...ver/tpg1/acls> cd ../luns 
/iscsi/iqn.20...ver/tpg1/luns> create /backstores/block/data
 
====设置本机IPSAN客户端访问地址及端口====
/iscsi/iqn.20...ver/tpg1/acls> cd ../portals/
/iscsi/iqn.20.../tpg1/portals> delete 0.0.0.0 3260
/iscsi/iqn.20.../tpg1/portals> create 192.168.100.131 3260
 
====保存配置退出====
/iscsi/iqn.20.../tpg1/portals> cd / 
/> saveconfig
/> exit
 
====重启target服务====
[root@server01 ~]# systemctl restart target
```

#### 防火墙放行服务

```
[root@server01 ~]# setenforce 0
[root@server01 ~]# firewall-cmd --permanent --add-service=iscsi-target
[root@server01 ~]# firewall-cmd --reload
```

### 客户端

#### 安装ISCSI客户端服务

```
[root@server02 ~]# dnf install iscsi-initiator-utils –y
[root@server02 ~]# systemctl start iscsi
```

设置客户端iscsi名称

```
[root@server02 ~]# vi /etc/iscsi/initiatorname.iscsi
 
====修改====
InitiatorName=iqn.2024-07.com.shaer-disk:client01
```

#### 发现远程服务器的共享

```
[root@server02 ~]# iscsiadm --mode discovery --type sendtargets --portal 192.168.100.131
192.168.100.131:3260,1 iqn.2024-07.com.shaer-disk:server
```

#### 连接共享

```
[root@server02 ~]# iscsiadm --mode node iqn.2024-07.com.shaer-disk:server --portal 192.168.100.131:3260 –login
 
====查看连接的设备====
[root@server02 ~]# lsblk 
NAME        MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
sda           8:0    0  20G  0 disk 
sr0          11:0    1   7G  0 rom  
nvme0n1     259:0    0  20G  0 disk 
├─nvme0n1p1 259:1    0   1G  0 part /boot
└─nvme0n1p2 259:2    0  19G  0 part 
  ├─cl-root 253:0    0  17G  0 lvm  /
  └─cl-swap 253:1    0   2G  0 lvm  [SWAP]
```

#### 分区格式化

创建分区(略)

```
[root@server02 ~]# fdisk /dev/sda  
...
[root@server02 ~]# mkfs.ext4 /dev/sda1
```

#### 自动挂载

```
[root@server02 ~]# vi /etc/fstab
 
====添加====
/dev/sda1               /opt/block1             ext4    _netdev         0 0
 
[root@server02 ~]# mount -a
[root@server02 ~]# df -h | grep sda1  
/dev/sda1             20G   45M   19G    1% /opt/block1
 
注:<options> 挂载时使用的参数 应为_netdev   代表其实从远端挂载，否则机器无法启动
```





## IPSAN多链路服务部署

#### 简介

> IPSAN的核心技术是iSCSI（Internet SCSI），即Internet SCSI，是IETF制订的一项标准。iSCSI技术将存储行业广泛应用的SCSI接口技术与IP网络技术相结合，可以在IP网络上构建SAN。简单地说，iSCSI就是在IP网络上运行SCSI协议的一种网络存储技术
> 单点故障在生产环境中是不被允许的，我们运维在设计架构的时候，如果无法解决单点故障，那么他设计的这个架构就无法满足高可用的需求，自然容灾性就无法谈起。同样我们在设计IPSAN架构的时候，也需要考虑单点故障的问题，因为一旦线路出现了问题，那么业务就会被中断了。这种问题我们是不能容忍的，这节课我就给大家说下如何实现IPSAN多链路共享。

### 服务端

#### 环境

| 设备             | 网卡1              | 网卡2             |
| ---------------- | ------------------ | ----------------- |
| Server01(服务端) | 192.168.100.131/24 | 192.168.50.131/24 |
| Server02(客户端) | 200.168.100.132/24 | 192.168.50.132/24 |

注:Server01(服务端)添加一块硬盘进行实验,下面我以nvme0n2(20G)为例子

#### 安装ISCSI服务

```
[root@server01 ~]# dnf install targetcli –y
[root@server01 ~]# systemctl enable --now target
```

#### 创建共享设备

```
[root@server01 ~]# targetcli 
targetcli shell version 2.1.fb49
Copyright 2011-2013 by Datera, Inc and others.
For help on commands, type 'help'.
 
/> ls 
o- / ....................................................................................... [...]
  o- backstores ............................................................................ [...]
  | o- block ................................................................ [Storage Objects: 0]
  | o- fileio ............................................................... [Storage Objects: 0]
  | o- pscsi ................................................................ [Storage Objects: 0]
  | o- ramdisk .............................................................. [Storage Objects: 0]
  o- iscsi .......................................................................... [Targets: 0]
  o- loopback ....................................................................... [Targets: 0]
 
====添加ISCSI后端存储====
/> cd /backstores/block 
/backstores/block> create data  /dev/nvme0n2
 
====设置IQN共享名称====
/backstores/block> cd /iscsi 
/iscsi> create iqn.2024-07.com.shaer-disk:server 
 
====设置访问IQN共享的设备名称====
/backstores/block> cd /iscsi 
/iscsi> create iqn.2024-07.com.shaer-disk:server 
/iscsi> cd iqn.2024-07.com.shaer-disk:server/tpg1/acls/
/iscsi/iqn.20...ver/tpg1/acls> create  iqn.2024-07.com.shaer-disk:client01
/iscsi/iqn.20...ver/tpg1/acls> cd ../luns 
/iscsi/iqn.20...ver/tpg1/luns> create /backstores/block/data
 
====设置本机IPSAN客户端访问地址及端口====
/iscsi/iqn.20...ver/tpg1/acls> cd ../portals/
/iscsi/iqn.20.../tpg1/portals> delete 0.0.0.0 3260
/iscsi/iqn.20.../tpg1/portals> create 192.168.100.131 3260
/iscsi/iqn.20.../tpg1/portals> create 192.168.50.131 3260
 
====保存配置退出====
/iscsi/iqn.20.../tpg1/portals> cd / 
/> saveconfig
/> exit
 
====重启target服务====
[root@server01 ~]# systemctl restart target
```

#### 防火墙放行服务

```
[root@server01 ~]# setenforce 0
[root@server01 ~]# firewall-cmd --permanent --add-service=iscsi-target
[root@server01 ~]# firewall-cmd --reload
```

### 客户端

#### 安装ISCSI客户端服务

```
[root@server02 ~]# dnf install iscsi-initiator-utils –y
[root@server02 ~]# systemctl start iscsi
```

设置客户端iscsi名称

```
[root@server02 ~]# vi /etc/iscsi/initiatorname.iscsi
 
====修改====
InitiatorName=iqn.2024-07.com.shaer-disk:client01
```

#### 发现远程服务器的共享

```
[root@server02 ~]# iscsiadm --mode discovery --type sendtargets --portal 192.168.100.131
192.168.100.131:3260,1 iqn.2024-07.com.shaer-disk:server
```

#### 连接共享

```
[root@server02 ~]# iscsiadm --mode node iqn.2024-07.com.shaer-disk:server --portal 192.168.100.131:3260 –login
 
====查看连接的设备====
[root@server02 ~]# lsblk 
NAME        MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
sda           8:0    0  20G  0 disk 
sdb           8:16   0  20G  0 disk 
sr0          11:0    1   7G  0 rom  
nvme0n1     259:0    0  20G  0 disk 
├─nvme0n1p1 259:1    0   1G  0 part /boot
└─nvme0n1p2 259:2    0  19G  0 part 
  ├─cl-root 253:0    0  17G  0 lvm  /
  └─cl-swap 253:1    0   2G  0 lvm  [SWAP]
注:其实这两块磁盘是一块盘，表示成两块的原因是我通过两个线路连接了两次，所以看到是多了两块。
   分区格式化的时候只需操作一个比如/dev/sdb就行了，另一个刷一下分区表就行了
```

#### 分区格式化

创建分区(略)

```
[root@server02 ~]# fdisk /dev/sda  
...
[root@server02 ~]# partprobe /dev/sdb
[root@server02 ~]# mkfs.ext4 /dev/sda1
 
注:分区格式化，注意格式化不要用xfs,不支持多路径，我们用的是ext4。
```

#### 安装多路径软件

#### 默认没有配置文件/etc/multipath.conf,需要复制例子配置文件

```
[root@server02 ~]# dnf install device-mapper-multipath-libs –y
[root@server02 ~]# cp /usr/share/doc/device-mapper-multipath/multipath.conf /etc/
[root@server02 ~]# systemctl start multipathd
 
====查看多路径挂载====
[root@server02 ~]# multipath -ll                
mpatha (360014056c0b0ad2d365480093dc3929d) dm-2 LIO-ORG,data
size=20G features='0' hwhandler='1 alua' wp=rw
|-+- policy='service-time 0' prio=50 status=active
| `- 2:0:0:0 sda     8:0   active ready running
`-+- policy='service-time 0' prio=50 status=enabled
  `- 3:0:0:0 sdb     8:16  active ready running
 
 
注:360014056c0b0ad2d365480093dc3929d  （远程存储设备id）
默认提供的是AB备份线路，不是负载均衡线路,我们可以通过修改配置文件来设置为负载均衡线路
 
 
====取消注释====
[root@server02 ~]# vi /etc/multipath.conf
multipaths {
    multipath {
        wwid            360014056c0b0ad2d365480093dc3929d    填写远程存储设备id
        alias           iscsi-disk     别名随便起，有意义就行
        path_grouping_policy    multibus            多路径组策略
        path_selector        "round-robin 0"     负载均衡模式
        failback        manual
        rr_weight        priorities              按优先级轮询
        no_path_retry        5                   重试时间5s
    }
    multipath {
        wwid            1DEC_____321816758474
        alias            red
    }
}
 
====重启服务查看====
[root@server02 ~]# systemctl restart multipathd.service
[root@server02 ~]# multipath -ll               
Jul 25 11:27:51 | /etc/multipath.conf line 65, invalid keyword: path_checker
iscsi-disk (360014056c0b0ad2d365480093dc3929d) dm-2 LIO-ORG,data
size=20G features='1 queue_if_no_path' hwhandler='1 alua' wp=rw
`-+- policy='round-robin 0' prio=50 status=enabled
  |- 2:0:0:0 sda     8:0   active ready running
  `- 3:0:0:0 sdb     8:16  active ready running
发现已经变成负载均衡模式了。
 
 
[root@server02 ~]# lsblk 
NAME            MAJ:MIN RM SIZE RO TYPE  MOUNTPOINT
sda               8:0    0  20G  0 disk  
└─iscsi-disk    253:2    0  20G  0 mpath 
  └─iscsi-disk1 253:3    0  20G  0 part  /opt/block1
sdb               8:16   0  20G  0 disk  
└─iscsi-disk    253:2    0  20G  0 mpath 
  └─iscsi-disk1 253:3    0  20G  0 part  /opt/block1
sr0              11:0    1   7G  0 rom   /media
nvme0n1         259:0    0  20G  0 disk  
├─nvme0n1p1     259:1    0   1G  0 part  /boot
└─nvme0n1p2     259:2    0  19G  0 part  
  ├─cl-root     253:0    0  17G  0 lvm   /
  └─cl-swap     253:1    0   2G  0 lvm   [SWAP
原来显示的两块设备也变成了一个了
```

#### 自动挂载

```
[root@server02 ~]# vi /etc/fstab
 
====添加====
/dev/mapper/iscsi-disk1              /opt/block1             ext4    _netdev         0 0
 
[root@server02 ~]# mount -a
[root@server02 ~]# df -h | grep block1 
/dev/mapper/iscsi-disk1   20G   45M   19G    1% /opt/block1
 
 
注:<options> 挂载时使用的参数 应为_netdev   代表其实从远端挂载，否则机器无法启动
```

#### 测试

#### 将其中一个网卡关掉还能写入即为成功。







