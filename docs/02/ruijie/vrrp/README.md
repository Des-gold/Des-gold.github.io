# VRRP实验

### 简介

> 虚拟路由冗余协议(Virtual Router Redundancy Protocol，简称VRRP)是由IETF提出的解决局域网中配置静态网关出现单点失效现象的路由协议，1998年已推出正式的RFC2338协议标准。VRRP广泛应用在边缘网络中，它的设计目标是支持特定情况下IP数据流量失败转移不会引起混乱，允许主机使用单路由器，以及即使在实际第一跳路由器使用失败的情形下仍能够维护路由器间的连通性。

### 实验拓扑

![](https://s3.bmp.ovh/imgs/2024/10/10/0dfeb6b6e805719a.png)

### 实验需求

1.创建`VLAN`，为保证可靠性采用`VRRP`对网关进行冗余。

2.`VLAN10`能访问`VLAN20`

### 配置

```
SW:
Switch#configure  terminal 
Switch(config)#hostname SW 
SW(config)#vlan 10
SW(config-vlan)#vlan 20 
SW(config)#int range gi 0/0-1 
SW(config-if-range)#sw mode trunk 
SW(config-if-range)#sw trunk allowed vlan add 10,20
SW(config-vlan)#int gi 0/2 
SW(config-if-GigabitEthernet 0/2)#sw acc vlan 10 
SW(config-if-GigabitEthernet 0/2)#int gi 0/3 
SW(config-if-GigabitEthernet 0/3)#sw acc vlan 30 
 
RSR1:
Router>enable 
Router#config ter
Router#config terminal 
Router(config)#hostname RSR1
RSR1(config)#vlan 10 
RSR1(config-vlan)#vlan 20 
RSR1(config)#int range gi 0/0-1 
RSR1(config-if-range)#switchport 
RSR1(config-if-range)#sw mode trunk
RSR1(config-if-range)#sw trunk vlan
RSR1(config-if-range)#sw trunk allowed vlan add 10,20
 
RSR1(config-vlan)#int vlan 10 
RSR1(config-if-VLAN 10)#ip add 192.168.10.252 24 
RSR1(config-if-VLAN 10)#vrrp 1 ip 192.168.10.254 
RSR1(config-if-VLAN 10)#vrrp 1 priority 120
RSR1(config-if-VLAN 10)#int vlan 20 
RSR1(config-if-VLAN 20)#ip add 192.168.20.252 24 
RSR1(config-if-VLAN 20)#vrrp 2 ip 192.168.20.254 
RSR1(config-if-VLAN 20)#exit 
 

RSR2:
Router>enable 
Router#config ter
Router#config terminal 
Router(config)#hostname RSR2
RSR2(config)#vlan 10 
RSR2(config-vlan)#vlan 20 
RSR2(config)#int range gi 0/0-1 
RSR2(config-if-range)#switchport 
RSR2(config-if-range)#sw mode trunk
RSR2(config-if-range)#sw trunk vlan
RSR2(config-if-range)#sw trunk allowed vlan add 10,20
 
RSR2(config-vlan)#int vlan 10 
RSR2(config-if-VLAN 10)#ip add 192.168.10.253 24 
RSR2(config-if-VLAN 10)#vrrp 1 ip 192.168.10.254 
RSR2(config-if-VLAN 10)#int vlan 20 
RSR2(config-if-VLAN 20)#ip add 192.168.20.253 24 
RSR2(config-if-VLAN 20)#vrrp 2 ip 192.168.20.254 
RSR2(config-if-VLAN 20)#vrrp 2 priority 120
RSR2(config-if-VLAN 20)#exit 
```

#### 测试

1.查看VRRP

```
RSR1#show vrrp brief 
Interface        Grp  Pri   timer   Own  Pre   State   Master addr          Group addr                  
VLAN 10          1    120   3.53    -    P     Master  192.168.10.252       192.168.10.254   
VLAN 20          2    100   3.60    -    P     Backup  192.168.20.253       192.168.20.254  
 
RSR2#show vrrp brief 
Interface        Grp  Pri   timer   Own  Pre   State   Master addr            Group addr 
VLAN 10          1    100   3.60    -    P     Backup  192.168.10.252       192.168.10.254
VLAN 20          2    120   3.53    -    P     Master  192.168.20.253       192.168.20.254 
```

2.VPC4访问VPC4

```
VPCS> ping 192.168.10.254
 
84 bytes from 192.168.10.254 icmp_seq=1 ttl=64 time=685.337 ms
84 bytes from 192.168.10.254 icmp_seq=2 ttl=64 time=68.526 ms
84 bytes from 192.168.10.254 icmp_seq=3 ttl=64 time=12.060 ms
84 bytes from 192.168.10.254 icmp_seq=4 ttl=64 time=46.608 ms
192.168.10.254 icmp_seq=5 timeout
 
VPCS> ping 192.168.20.254
 
84 bytes from 192.168.20.254 icmp_seq=1 ttl=64 time=582.272 ms
84 bytes from 192.168.20.254 icmp_seq=2 ttl=64 time=170.265 ms
192.168.20.254 icmp_seq=3 timeout
192.168.20.254 icmp_seq=4 timeout
84 bytes from 192.168.20.254 icmp_seq=5 ttl=64 time=51.868 ms
 
VPCS> ping 192.168.20.1
 
192.168.20.1 icmp_seq=1 timeout
84 bytes from 192.168.20.1 icmp_seq=2 ttl=63 time=37.425 ms
84 bytes from 192.168.20.1 icmp_seq=3 ttl=63 time=51.257 ms
192.168.20.1 icmp_seq=4 timeout
192.168.20.1 icmp_seq=5 timeout
注:丢包是正常现象
```