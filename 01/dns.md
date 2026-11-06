## DNS安装与正反向解析

#### 简介

> 域名系统（Domain Name System，DNS）是把域名和 IP 地址相互映射的分布式数据库，使用 UDP 53 端口。解析分两个方向：正向由域名查 IP（A 记录），反向由 IP 查域名（PTR 记录）。
> BIND 是 Linux 上最常用的 DNS 服务，主配置文件 /etc/named.conf 管全局参数，/etc/named.rfc1912.zones 管区域声明，区域数据文件放 /var/named/。

### 1.配置步骤

```bash
(1)安装bind并修改主配置文件
[root@server01 ~]# dnf install bind -y
[root@server01 ~]# vi /etc/named.conf

options {
        listen-on port 53 { any; };              # 改为any
        listen-on-v6 port 53 { any; };           # 改为any
        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
        secroots-file   "/var/named/data/named.secroots";
        recursing-file  "/var/named/data/named.recursing";
        allow-query     { any; };                # 改为any
...

# 注:listen-on 决定监听哪些地址，allow-query 决定允许哪些客户端查询，默认只放行 127.0.0.1，不改外部一律查不到

(2)添加正向区域
[root@server01 ~]# vi /etc/named.rfc1912.zones

# 添加
zone "test.com" IN {
        type master;
        file "test.empty";
        allow-update { none; };
};

(3)编写正向区域文件，用 named.empty 做模板保留属组
[root@server01 ~]# cd /var/named/
[root@server01 named]# cp -a named.empty test.empty
[root@server01 named]# vi test.empty

# 修改
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

(4)添加反向区域
[root@server01 ~]# vi /etc/named.rfc1912.zones

# 添加
zone "100.168.192.in-addr.arpa" IN {
        type master;
        file "test.loopback";
        allow-update { none; };
};

(5)编写反向区域文件
[root@server01 named]# vi test.loopback

# 修改
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

# 注:反向区域名是网段倒着写，192.168.100.0/24 对应 100.168.192.in-addr.arpa；区域文件必须属组 named，否则 named 读不到直接报 permission denied

(6)启动服务并放行
[root@server01 named]# systemctl start named
[root@server01 named]# setenforce 0
[root@server01 named]# firewall-cmd --permanent --add-service=dns
[root@server01 named]# firewall-cmd --reload
```

### 2.验证配置

```bash
测试端将DNS设置为服务端的IP地址，需要安装测试工具（图形化免安装）
[root@server01 named]# dnf install bind-utils -y
[root@server01 named]# nslookup www.test.com
Server:         192.168.100.131
Address:        192.168.100.131#53

Name:   www.test.com
Address: 192.168.100.131

[root@server01 named]# nslookup 192.168.100.131
131.100.168.192.in-addr.arpa    name = www.test.com.
```

## DNS主从服务器配置

#### 简介

> 主从是两个独立的 named 各自持有同一份区域数据。主服务器负责写入和应答，从服务器定时用 SOA 里的 refresh 时间向主服务器拉取，主挂掉后从仍能继续解析，同时分担查询压力。
> 从服务器的区域类型写 slave，不手写区域文件，启动后自动同步到 /var/named/slaves/ 下。

### 1.环境

|   主机   |      地址       |    角色     |
| :------: | :-------------: | :---------: |
| Server01 | 192.168.100.131 | 主DNS服务器 |
| Server02 | 192.168.100.132 | 从DNS服务器 |

### 2.配置步骤

```bash
主DNS服务器配置步骤如上述（略）

(1)从服务器安装bind并修改主配置文件，改法与主服务器相同
[root@server02 ~]# dnf install bind -y
[root@server02 ~]# vi /etc/named.conf

options {
        listen-on port 53 { any; };              # 改为any
        listen-on-v6 port 53 { any; };           # 改为any
        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
        secroots-file   "/var/named/data/named.secroots";
        recursing-file  "/var/named/data/named.recursing";
        allow-query     { any; };                # 改为any
...

(2)声明从区域，type 改为 slave、masters 指向主服务器
[root@server02 ~]# vi /etc/named.rfc1912.zones

# 添加
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

# 注:从服务器不手写区域文件，启动后自动从 masters 同步到 /var/named/slaves/ 下，file 路径必须带 slaves/ 前缀

(3)启动服务并放行
[root@server02 ~]# systemctl restart named
[root@server02 ~]# setenforce 0
[root@server02 ~]# firewall-cmd --permanent --add-service=dns
[root@server02 ~]# firewall-cmd --reload
```

### 3.验证配置

```bash
# 1.查看是否同步区域文件
[root@server02 ~]# ls /var/named/slaves
test.loopback  test.empty

# 2.将主DNS服务器的服务暂停，将测试端DNS设置为从服务器进行解析
[root@server02 ~]# nslookup ns.test.com
Server:         192.168.100.132
Address:        192.168.100.132#53

Name:   ns.test.com
Address: 192.168.100.131
```
