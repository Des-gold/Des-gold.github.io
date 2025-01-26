## DNS的安装及配置

#### 简介

> 域名系统（英文：Domain Name System，缩写：DNS）是互联网的一项服务。它作为将域名和IP地址相互映射的一个分布式数据库，能够使人更方便地访问互联网。DNS使用UDP端口53。当前，对于每一级域名长度的限制是63个字符，域名总长度则不能超过253个字符。

#### 安装服务及修改配置

```
[root@server01 ~]# dnf install bind –y
[root@server01 ~]# vi /etc/named.conf
 
options {
        listen-on port 53 { any; };              #改为any
        listen-on-v6 port 53 { any; };             #改为any
        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
        secroots-file   "/var/named/data/named.secroots";
        recursing-file  "/var/named/data/named.recursing";
        allow-query     { any; };             #改为any
... 
```

#### 正向解析

```
[root@server01 ~]# vi /etc/named.rfc1912.zones
 
====添加====
 
zone "test.com" IN {
        type master;
        file "test.empty";
        allow-update { none; };
};
 
 
[root@server01 ~]# cd /var/named/
[root@server01 named]# cp -a named.empty  test.empty
[root@server01 named]# vi test.empty  
 
====修改====
$TTL 3H
@       IN SOA  test.com. rname.invalid. (
                                        0       ; serial
                                        1D      ; refresh
                                        1H      ; retry
                                        1W      ; expire
                                        3H )    ; minimum
        NS      ns.test.com.
ns      A       192.168.100.131
www     A       192.168.100.131
```

#### 反向解析

```
[root@server01 ~]# vi /etc/named.rfc1912.zones
 
==== 添加====
zone "100.168.192.in-addr.arpa" IN {
        type master;
        file "test.loopback";
        allow-update { none; };
};
 
 
 
[root@server01 named]# vi /etc/named.rfc1912.zones
[root@server01 named]# vi test.loopback
 
====修改====
$TTL 1D
@       IN SOA  test.com. rname.invalid. (
                                        0       ; serial
                                        1D      ; refresh
                                        1H      ; retry
                                        1W      ; expire
                                        3H )    ; minimum
        NS      ns.test.com.
ns      A       192.168.100.131
131     PTR     www.test.com.
```

#### 启动、防火墙放行服务

```
[root@server01 named]# systemctl start named
[root@server01 named]# setenforce 0
[root@server01 named]# firewall-cmd --permanent --add-service=dns
[root@server01 named]# firewall-cmd –reload
```

#### 测试

测试端将DNS设置为服务端的IP地址，需要安装测试工具(图形化免安装)

```
[root@server01 named]# dnf install bind-utils -y 
[root@server01 named]# nslookup www.test.com
Server:         192.168.100.131
Address:        192.168.100.131#53
 
Name:   www.test.com
Address: 192.168.100.131
 
[root@server01 named]# nslookup 192.168.100.131
131.100.168.192.in-addr.arpa    name = www.test.com.
```





## DNS的主从服务器配置

|   主机   |      地址       |    角色     |
| :------: | :-------------: | :---------: |
| Server01 | 192.168.100.131 | 主DNS服务器 |
| Server02 | 192.168.100.132 | 从DNS服务器 |

#### 主DNS服务器配置

主DNS服务器配置步骤如上述（略）

#### 从DNS服务器配置

#### 安装服务及修改配置

```
[root@server02 ~]# dnf instal bind –y
[root@server02 ~]# vi /etc/named.conf
 
options {
        listen-on port 53 { any; };              #改为any
        listen-on-v6 port 53 { any; };             #改为any
        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
        secroots-file   "/var/named/data/named.secroots";
        recursing-file  "/var/named/data/named.recursing";
        allow-query     { any; };             #改为any
... 
 
 
[root@server02 ~]# vi /etc/named.rfc1912.zones
 
====添加====
zone "test.com" IN {
        type slave;     
        file "slaves/test.empty";
        masters { 192.168.100.131; };
};
 
zone "100.168.192.in-addr.arpa" IN {
        type slave;
        file "slaves/test.loopback";
        masters { 192.168.100.131; };
};
```

#### 启动服务防火墙放行服务

```
[root@server02 ~]# systemctl restart named
[root@server02 ~]# setenforce 0
[root@server02 ~]# firewall-cmd --permanent --add-service=dns
[root@server02 ~]# firewall-cmd –reload
```

#### 测试

1.查看是否同步区域文件

```
[root@server02 ~]# ls /var/named/slaves
stest.loopback  test.empty
```

2.将主DNS服务器的服务暂停，将测试端DNS设置为从服务器进行解析

```
[root@server02 ~]# nslookup ns.test.com
Server:         192.168.100.132
Address:        192.168.100.132#53
 
Name:   ns.test.com
Address: 192.168.100.131
```

