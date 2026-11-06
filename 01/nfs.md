## NFS服务端配置

#### 简介

> NFS是基于UDP/IP协议的应用，其实现主要是采用远程过程调用RPC机制，RPC提供了一组与机器、操作系统以及底层传送协议无关的存取远程文件的操作。RPC采用了XDR的支持。XDR是一种与机器无关的数据描述编码的协议，他以独立与任意机器体系结构的格式对网上传送的数据进行编码和解码，支持在异构系统之间数据的传送。

### 1.安装服务

```bash
[root@server01 ~]# dnf install nfs-utils -y
[root@server01 ~]# dnf install rpcbind -y
```

### 2.创建共享目录及配置

```bash
[root@server01 ~]# mkdir /test
[root@server01 ~]# chmod 777 /test
[root@server01 ~]# vi /etc/exports

# 添加
/test 192.168.100.0/24(rw)

[root@server01 ~]# exportfs -r

# 注:改完 /etc/exports 执行 exportfs -r 就重新加载，不用重启服务
```

### 3.启动服务、防火墙放行服务

```bash
[root@server01 ~]# systemctl restart rpcbind
[root@server01 ~]# systemctl restart nfs-server
[root@server01 ~]# setenforce 0
[root@server01 ~]# firewall-cmd --permanent --add-service=rpc-bind
[root@server01 ~]# firewall-cmd --permanent --add-service=nfs
[root@server01 ~]# firewall-cmd --permanent --add-service=mountd
[root@server01 ~]# firewall-cmd --reload

# 注:RHEL8 的服务单元叫 nfs-server，不是 nfs-utils；nfs-utils 只是包名，写 systemctl restart nfs-utils 会报 Unit not found
```

## NFS客户端访问

#### 简介

> 客户端不用装服务端，先确认能看到共享，再挂到本地目录用。Linux 用 mount，Windows 在资源管理器里输地址。

### 1.查看共享

```bash
[root@server02 ~]# showmount -e 192.168.100.131
Export list for 192.168.100.131:
/test 192.168.100.0/24
```

### 2.Windows客户端

此电脑 - 路径栏输入 \\192.168.100.131（NFS服务器端IP地址）

### 3.Linux客户端临时挂载

```bash
[root@server02 ~]# mkdir /test
[root@server02 ~]# mount -t nfs4 192.168.100.131:/test /test
[root@server02 ~]# df -h | grep test
192.168.100.131:/test   17G  4.2G   13G   25% /test
```

### 4.Linux客户端自动挂载

```bash
[root@server02 ~]# vi /etc/fstab

# 添加
192.168.100.131:/test   /test   nfs4    rw,sync,hard,intr       0 0

[root@server02 ~]# mount -a
[root@server02 ~]# df -h | grep test
192.168.100.131:/test   17G  4.2G   13G   25% /test
```

Windows客户端也可以用映射网络驱动器，此电脑 - 映射网络驱动器。
