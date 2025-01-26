## Linux DHCP配置

#### 简介

> 动态主机配置协议是一个局域网的网络协议。指的是由服务器控制一段IP地址范围，客户机登录服务器时就可以自动获得服务器分配的IP地址和子网掩码。担任DHCP服务器的计算机需要安装TCP/IP协议，并为其设置静态IP地址、子网掩码、默认网关等内容。

#### 安装

```
[root@server01 ~]# dnf install dhcp-server –y
```

#### 修改服务配置文件

```
[root@server01 ~]# vi /etc/dhcp/dhcpd.conf
 
====添加====
 
subnet 192.168.100.0 netmask 255.255.255.0
{
        range 192.168.100.100 192.168.100.200;
        option routers 192.168.100.1;
        option domain-name-servers 192.168.100.8;
        option domain-name "www.kim6.cn";
}
 
====保留地址====
 
host win7
{
        hardware ethernet 00:0c:29:c3:e2:78;
        fixed-address 192.168.100.10;
}
```

#### 启动服务、防火墙放行服务

```
[root@server01 ~]# systemctl enable --now dhcpd
[root@server01 ~]# firewall-cmd --permanent --add-service=dhcp
[root@server01 ~]# firewall-cmd –reload
```

#### 报错解决方法

```
[root@server01 ~]# dhcpd -t
====配置错误输出的内容====
Internet Systems Consortium DHCP Server 4.3.6
Copyright 2004-2017 Internet Systems Consortium.
All rights reserved.
For info, please visit https://www.isc.org/software/dhcp/
/etc/dhcp/dhcpd.conf line 15: semicolon expected.
        default-lease-time
         ^
/etc/dhcp/dhcpd.conf line 24: semicolon expected.
        harware ethernet
                 ^
Configuration file errors encountered -- exiting
This version of ISC DHCP is based on the release available
on ftp.isc.org. Features have been added and other changes
have been made to the base software release in order to make
it work better with this distribution.
Please report issues with this software via:
https://bugzilla.redhat.com/
exiting.
 
====配置正确输出的内容====
Internet Systems Consortium DHCP Server 4.3.6
Copyright 2004-2017 Internet Systems Consortium.
All rights reserved.
For info, please visit https://www.isc.org/software/dhcp/
ldap_gssapi_principal is not set,GSSAPI Authentication for LDAP will not be used
Not searching LDAP since ldap-server, ldap-port and ldap-base-dn were not specified in the config file
Config file: /etc/dhcp/dhcpd.conf
Database file: /var/lib/dhcpd/dhcpd.leases
PID file: /var/run/dhcpd.pid
Source compiled to use binary-leases
```

## DHCP中继配置

| 主机                 | 网卡           | IP              | GW          |
| -------------------- | -------------- | --------------- | ----------- |
| Server01(DHCP服务器) | VMnet2         | 172.16.42.10/24 | 172.16.42.2 |
| Relay(中继服务器)    | VMnet2         | 172.16.42.2/24  | 172.16.42.2 |
| VMnet3               | 192.168.1.2/24 | 192.168.1.2     |             |
| Client01             | VMnet3         | 自动获取        | 自动获取    |

#### 安装服务

分别在server01(DHCP服务器)和Relay(中继服务器)，安装dhcp-server、dhcp-relay服务

```
[root@server01 ~]# dnf install dhcp-server -y
[root@Relay ~]# dnf install dhcp-relay -y
```

#### 修改服务配置文件

配置DHCP作用域及添加路由

```
[root@server01 ~]# vi /etc/dhcp/dhcpd.conf
 
====写入以下内容====
 
subnet 172.16.42.0 netmask 255.255.255.0{
}
subnet 192.168.1.0 netmask 255.255.255.0{
        range 192.168.1.100 192.168.1.200;
        option routers 192.168.1.2;
        option domain-name-servers 192.168.1.2;
}
 
[root@server01 ~]# systemctl start dhcpd
[root@server01 ~]# ip route add 192.168.1.0/24  via 172.16.42.2
```

开启Relay(中继服务器)的ip转发模式

```
[root@relay ~]# echo net.ipv4.ip_forward = 1 >> /etc/sysctl.conf 
[root@relay ~]# sysctl -p 
net.ipv4.ip_forward = 1
```

在Relay(中继服务器)指定DHCP服务器

```
[root@relay ~]# dhcrelay 172.16.42.10
Dropped all unnecessary capabilities.
Internet Systems Consortium DHCP Relay Agent 4.3.6
Copyright 2004-2017 Internet Systems Consortium.
All rights reserved.
For info, please visit https://www.isc.org/software/dhcp/
Listening on LPF/ens224/00:0c:29:f3:42:db
Sending on   LPF/ens224/00:0c:29:f3:42:db
Listening on LPF/ens160/00:0c:29:f3:42:d1
Sending on   LPF/ens160/00:0c:29:f3:42:d1
Sending on   Socket/fallback
```
