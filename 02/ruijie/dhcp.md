## DHCP实验

#### 简介

> 动态主机配置协议是一个局域网的网络协议。指的是由服务器控制一段IP地址范围，客户机登录服务器时就可以自动获得服务器分配的IP地址和子网掩码。担任DHCP服务器的计算机需要安装TCP/IP协议，并为其设置静态IP地址、子网掩码、默认网关等内容。

#### 实验拓扑

![](https://s3.bmp.ovh/imgs/2024/10/10/f55a8aca310d2647.png)

#### 实验需求

主机需要自动获取地址，地址租期为3天,DNS地址和网关相同:`192.168.10.254`

#### 配置

```
SW2:
Switch>enable 
Switch#configure terminal 
Switch(config)#hostname SW2
SW2(config)#service dhcp 	//开启DHCP功能
SW2(config)#ip dhcp pool vlan10
SW2(dhcp-config)#network 192.168.10.0 255.255.255.0 
SW2(dhcp-config)#default-router  192.168.10.254 
SW2(dhcp-config)#dns-server 192.168.10.254 
SW2(dhcp-config)#lease 3 0 0 
SW2(dhcp-config)#vlan 10 
SW2(config-vlan)#int vlan 10 
SW2(config-if-VLAN 10)#ip add 192.168.10.254 24 
SW2(config)#int range gi 0/0-1 
SW2(config-if-range)#sw acc vlan 10
```

#### 测试

主机VPC3/VPC4是否能够获取地址

```
VPC3:
VPCS> ip dhcp 
DDORA IP 192.168.10.1/24 GW 192.168.10.254
 
VPCS> show ip 
 
NAME        : VPCS[1]
IP/MASK     : 192.168.10.1/24
GATEWAY     : 192.168.10.254
DNS         : 192.168.10.254  
DHCP SERVER : 192.168.10.254
DHCP LEASE  : 259165, 259200/129600/226800
MAC         : 00:50:79:66:68:03
LPORT       : 20000
RHOST:PORT  : 127.0.0.1:30000
MTU         : 1500
 
 
VPC4:
VPCS> ip dhcp 
DDORA IP 192.168.10.2/24 GW 192.168.10.254
 
VPCS> show ip 
 
NAME        : VPCS[1]
IP/MASK     : 192.168.10.2/24
GATEWAY     : 192.168.10.254
DNS         : 192.168.10.254  
DHCP SERVER : 192.168.10.254
DHCP LEASE  : 259151, 259200/129600/226800
MAC         : 00:50:79:66:68:04
LPORT       : 20000
RHOST:PORT  : 127.0.0.1:30000
MTU         : 1500
```

## DHCP Relay实验

#### 简介

> DHCP请求报文的目的IP地址为255.255.255.255，这种类型报文的转发局限于子网内。为了实现跨网段的动态IP地址分配，DHCP中继就产生了。DHCP中继将收到的DHCP请求报文以单播方式转发给DHCP服务器，同时将收到的DHCP响应报文转发给DHCP客户端。DHCP中继相当于一个转发站，负责沟通位于不同网段的DHCP客户端和DHCP服务器，即转发客户端DHCP请求报文、转发服务端DHCP应答报文。这样就实现了只要安装一个DHCP服务器，就可以实现对多个网段的动态IP管理，即Client—Relay—Server模式的DHCP动态IP管理。

#### 实验拓扑

![](https://s3.bmp.ovh/imgs/2024/10/10/ac1279d4884a07b2.png)

#### 实验需求

汇聚交换机为用户的网关，开启DHCP relay功能。核心交换机为网络核心设备，开启DHCP Server功能，汇聚交换机与核心交换机使用三层口互联

#### 配置

```
SW1:
Switch>enable 
Switch#configure terminal 
SW1(config)#hostname SW1
Switch(config)#int gi 0/0 
SW1(config-if-GigabitEthernet 0/0)#no SW1port 
SW1(config-if-GigabitEthernet 0/0)#ip add 172.16.1.1 30 
SW1(config-if-GigabitEthernet 0/0)#vlan 10 
SW1(config-if-VLAN 10)#service dhcp 
SW1(config)#ip dhcp pool vlan10 
SW1(dhcp-config)#network 192.168.10.0 255.255.255.0 
SW1(dhcp-config)#default-router 192.168.10.254 
SW1(dhcp-config)#dns-server 192.168.10.254 
SW1(dhcp-config)#ip route 192.168.10.0 255.255.255.0 172.16.1.2 
 
 
SW2:
Switch>enable 
Switch#configure terminal 
SW2(config)#hostname SW2
SW2(config)#service dhcp
SW2(config)#int gi 0/2 
SW2(config-if-GigabitEthernet 0/2)#no SW2port 
SW2(config-if-GigabitEthernet 0/2)#ip add 172.16.1.2 30 
SW2(config-if-GigabitEthernet 0/2)#vlan 10 
SW2(config-vlan)#int vlan 10 
SW2(config-if-VLAN 10)#ip add 192.168.10.254 24 
SW2(config-if-VLAN 10)#ip route 0.0.0.0 0.0.0.0 172.16.1.1
SW2(config)#ip helper-address 172.16.1.1     //开启DHCP relay功能
SW2(config)#int range gi 0/0-1 
SW2(config-if-range)#sw acc vlan 10 
```

#### 测试

1.主机VPC3/VPC4是否能够获取地址

```
VPC3:
VPCS> ip dhcp 
DDORA IP 192.168.10.3/24 GW 192.168.10.254
 
VPC4:
VPCS> ip dhcp 
DDORA IP 192.168.10.4/24 GW 192.168.10.254
```

2.在DHCP服务器查询有哪些终端

```
SW1#show ip dhcp binding 
 
Total number of clients   : 2
Expired clients           : 0
Running clients           : 2
 
IP address        Hardware address       Lease expiration            Type
192.168.10.4      0050.7966.6804         000 days 23 hours 59 mins   Automatic
192.168.10.3      0050.7966.6803         000 days 23 hours 59 mins   Automatic
```

## DHCP Snooping实验

#### 简介

> 意为DHCP 窥探，在一次PC动态获取IP地址的过程中，通过对Client和服务器之间的DHCP交互报文进行窥探，实现对用户的监控，同时DHCP Snooping起到一个DHCP 报文过滤的功能，通过合理的配置实现对非法服务器的过滤，防止用户端获取到非法DHCP服务器提供的地址而无法上网。

#### 实验拓扑

![](https://s3.bmp.ovh/imgs/2024/10/10/ac1279d4884a07b2.png)

#### 实验需求

换机下联PC使用动态DHCP获取IP地址，为了防止内网有用户接入非法dhcp服务器，如自带的无线小路由器等，导致正常用户获取到错误的地址而上不了网或地址冲突，要实施DHCP Snooping（DHCP监听）功能。

#### 配置

```
SW1:
Switch>enable 
Switch#configure terminal 
SW1(config)#hostname SW1
SW1(config)#service dhcp 		//开启DHCP功能
SW1(config)#int vlan 1 
SW1(config-if-VLAN 1)#ip add 10.1.1.254 24 
SW1(config-if-VLAN 1)#ip dhcp pool vlan1
SW1(dhcp-config)#network 10.1.1.0 255.255.255.0  
SW1(dhcp-config)#default-router 10.1.1.254 
SW1(dhcp-config)#dns-server 10.1.1.254 
 
 
SW2:
Switch>enable 
Switch#configure terminal 
SW2(config)#hostname SW2
SW2(config)#ip dhcp snooping 			//开启DHCP snooping功能 
SW2(config)#int gi 0/2 
SW2(config-if-GigabitEthernet 0/2)#ip dhcp snooping trust   	 //连接DHCP服务器的接口配置为可信任口
 
注:开启DHCP snooping的交换机所有接口缺省为untrust口，交换机只转发从trust口收到的DHCP响应报文（offer、ACK） 
```

#### 测试

1.主机VPC3/VPC4是否能够获取地址

```
VPC3:
VPCS> ip dhcp 
DDORA IP 10.1.1.1/24 GW 10.1.1.254
 
VPC4:
VPCS> ip dhcp 
DDORA IP 10.1.1.2/24 GW 10.1.1.254
```

2.查看DHCP Snooping表及设置情况

```
SW2(config)#show ip  dhcp snooping  binding 
 
Total number of bindings: 2 
 
NO.   MACADDRESS         IPADDRESS       LEASE(SEC)   TYPE          VLAN  INTERFACE
----- ------------------ --------------- ------------ ------------- ----- --------------------
1     0050.7966.6804     10.1.1.2        86270        DHCP-Snooping 1     GigabitEthernet 0/1 
2     0050.7966.6803     10.1.1.1        86247        DHCP-Snooping 1     GigabitEthernet 0/0 
 
 
 
 
SW2(config)#show ip dhcp snooping 
...
 
Interface                       Trusted         Rate limit (pps)
------------------------        -------         ----------------
GigabitEthernet 0/2             YES             unlimited       
Default                         No              unlimited 
```