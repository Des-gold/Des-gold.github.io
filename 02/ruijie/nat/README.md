# NAT实验

#### 简介

> NAT旨在通过将一个外部 IP 地址和端口映射到更大的内部 IP 地址集来转换 IP 地址。 基本上，NAT 使用流量表将流量从一个外部（主机）IP 地址和端口号路由到与网络上的终结点关联的正确内部 IP 地址。

#### 实验拓扑

![](https://s3.bmp.ovh/imgs/2024/10/06/3c50a6462c0948ac.png)

#### 实验需求

1.内网设备地址获取使用自动获取

2.内网能访问外网地址`100.1.1.1`

#### 配置

#### 基础配置

```
Switch>enable 
Switch#configure terminal 
Switch(config)#hostname Ruijie-SW2
Ruijie-SW2(config)#int loopback 0
Ruijie-SW2(config-if-Loopback 0)#ip add 100.1.1.1 24 
Ruijie-SW2(config)#int gi 0/0 
Ruijie-SW2(config-if-GigabitEthernet 0/0)#no switchport        
Ruijie-SW2(config-if-GigabitEthernet 0/0)#ip add 10.1.1.1 24
 
 
Router>enable 
Router#configure terminal 
Router(config)#int gi 0/1 
Router(config-if-GigabitEthernet 0/1)#ip add 10.1.1.2 24 
Router(config-if-GigabitEthernet 0/1)#exit 
Router(config)#ip route 0.0.0.0 0.0.0.0 10.1.1.1  
 
 
Switch>enable 
Switch#configure terminal 
Switch(config)#hostname SW  
SW(config)#vlan 10 
SW(config-vlan)#int vlan 10 
SW(config-if-VLAN 10)#ip add 192.168.1.253 24 
SW(config)#int gi 0/0 
SW(config-if-GigabitEthernet 0/0)#switchport access vlan 10
SW(config-if-GigabitEthernet 0/0)#exit 
SW(config)#int range gi 0/1-2 
SW(config-if-range)#switchport acc
SW(config-if-range)#switchport access vlan 10 
 
 
SW(config)#service dhcp 
SW(config)#ip dhcp pool test   
SW(dhcp-config)#network 192.168.1.0 255.255.255.0 
SW(dhcp-config)#default-router 192.168.1.254 
SW(dhcp-config)#dns-server 192.168.1.254 
SW(dhcp-config)#exit 
```

### 静态NAT

把内部主机的ip地址一对一的映射成公网ip地址，或者 内部主机的ip地址+端口 一对一的映射成 公网ip地址+端口。通常用在需要将内部主机映射成公网地址，或者内部服务器的某个端口映射成公网地址的某个端口，达到通过访问公网地址或者公网地址+端口号来访问内部服务器的目的。

```
Router(config)#int gi 0/1 
Router(config-if-GigabitEthernet 0/1)#ip nat outside 
Router(config-if-GigabitEthernet 0/1)#int gi 0/0 
Router(config-if-GigabitEthernet 0/0)#ip nat inside 
Router(config-if-GigabitEthernet 0/0)#exit
 
Router(config)#ip nat inside source static 192.168.1.2 10.1.1.2 
 
注:
ip nat inside 和 ip nat outside：定义内部和外部接口
ip nat inside source static：将内部IP地址192.168.1.10映射到外部IP地址10.1.1.2。
```

### 动态NAT

公网地址池的IP地址+端口号来对应和区别各个数据流进行网络地址转换，以达到多内部主机通过少量公网IP地址来访问外部网络的目的。通常用在有多个1个出口有多个公网ip地址的情况

```
Router(config)#int gi 0/1 
Router(config-if-GigabitEthernet 0/1)#ip nat outside 
Router(config-if-GigabitEthernet 0/1)#int gi 0/0 
Router(config-if-GigabitEthernet 0/0)#ip nat inside 
Router(config-if-GigabitEthernet 0/0)#exit
 
Router(config)#ip access-list standard 10 
Router(config-std-nacl)#5 permit 192.168.1.0 0.0.0.255 
 
 
Router(config)#ip nat pool natpool 10.1.1.2 10.1.1.3 netmask 255.255.255.0 
Router(config)#ip nat inside source list 10 pool natpool 
 
注:
ip nat pool：定义一个公共IP地址池。
access-list：定义允许NAT的内部IP地址范围。
ip nat inside source list：将匹配ACL的内部IP地址转换为地址池中的公共IP地址。
```

### PAT

端口地址转换，又叫网络地址端口转换（NAPT），通过 外网接口的IP地址+端口号来对应和区别各个数据流进行网络地址转换，以达到多内部主机通过外网接口的IP地址来访问外部网络的目的。通常用在只有一个公网地址时的场景。

```
Router(config)#int gi 0/1 
Router(config-if-GigabitEthernet 0/1)#ip nat outside 
Router(config-if-GigabitEthernet 0/1)#int gi 0/0 
Router(config-if-GigabitEthernet 0/0)#ip nat inside 
Router(config-if-GigabitEthernet 0/0)#exit
 
Router(config)#ip access-list standard 10 
Router(config-std-nacl)#5 permit 192.168.1.0 0.0.0.255 
 
Router(config)# ip nat inside source list 100 interface GigabitEthernet0/1 overload
 
注:ip nat inside source list ... overload：将匹配ACL的内部IP地址转换为外部接口的公共IP地址，同时使用不同的端口号实现过载。
```

#### NAT 配置验证

```
显示NAT转换条目
Ruijie# show ip nat translations
 
显示NAT统计信息
Ruijie# show ip nat statistics
```

#### 测试

使用内网VPCS或Server访问外网

```
VPCS> ping 100.1.1.1
 
84 bytes from 100.1.1.1 icmp_seq=1 ttl=63 time=7.715 ms
84 bytes from 100.1.1.1 icmp_seq=2 ttl=63 time=5.748 ms
84 bytes from 100.1.1.1 icmp_seq=3 ttl=63 time=4.488 ms
84 bytes from 100.1.1.1 icmp_seq=4 ttl=63 time=4.346 ms
84 bytes from 100.1.1.1 icmp_seq=5 ttl=63 
```