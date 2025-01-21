# NFS配置

### 简介

> NFS是基于UDP/IP协议的应用，其实现主要是采用远程过程调用RPC机制，RPC提供了一组与机器、操作系统以及底层传送协议无关的存取远程文件的操作。RPC采用了XDR的支持。XDR是一种与机器无关的数据描述编码的协议，他以独立与任意机器体系结构的格式对网上传送的数据进行编码和解码，支持在异构系统之间数据的传送。

### 安装服务

```
[root@server01 ~]# dnf install nfs-utils –y
[root@server01 ~]# dnf install rpcbind –y
```

### 创建 NFS 共享及配置

```
[root@server01 ~]# mkdir /test
[root@server01 ~]# chmod 777 /test
[root@server01 ~]# vi /etc/exports
 
====添加====
/test 192.168.100.0/24(rw)
 
[root@server01 ~]# exportfs –r
```

### 启动服务防火墙放行服务

```
[root@server01 ~]# systemctl restart rpcbind
[root@server01 ~]# systemctl restart nfs-utils
[root@server01 ~]# setenforce 0
[root@server01 ~]#firewall-cmd --permanent --add-service=rpc-bind
[root@server01 ~]#firewall-cmd --permanent --add-service=nfs
[root@server01 ~]#firewall-cmd --permanent --add-service=mountd
[root@server01 ~]#firewall-cmd --reload
```

## 测试

### Linxu客户端

```
[root@server02 ~]#showmount -e 192.168.100.131
Export list for 192.168.100.131:
/test 192.168.100.0/24
```

### Windows客户端

此电脑-路径栏输入\\192.168.100.131(NFS服务器端IP地址)

## 挂载

### 临时挂载

Linux客户端

```
[root@server02 ~]#mkdir /test
[root@server02 ~]#mount -t nfs4 192.168.100.131:/test /test
[root@server02 ~]# df -h | grep test
192.168.100.131:/test   17G  4.2G   13G   25% /test
```

### 自动挂载

Linux客户端

```
[root@server02 ~]# vi /etc/fstab 
 
====添加====
192.168.100.131:/test   /test   nfs4    rw,sync,hard,intr       0 0
 
[root@server02 ~]# mount -a 
[root@server02 ~]# df -h | grep test
192.168.100.131:/test   17G  4.2G   13G   25% /test
```

Windows客户端

此电脑-映射网络驱动器