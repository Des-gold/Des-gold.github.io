# GRE over IPsec实验

### 简介

> GRE over IPSec可利用GRE和IPSec的优势，通过GRE将组播、广播和非IP报文封装成普通的IP报文，通过IPSec为封装后的IP报文提供安全地通信，进而可以提供在总部和分支之间安全地传送广播、组播的业务，例如视频会议或动态路由协议消息等。当网关之间采用GRE over IPSec连接时，先进行GRE封装，再进行IPSec封装。GRE over IPSec使用的封装模式为可以是隧道模式也可以是传输模式。因为隧道模式跟传输模式相比增加了IPSec头，导致报文长度更长，更容易导致分片，所以推荐采用传输模式GRE over IPSec。

### 实验拓扑

[![img](http://localhost/lib/exe/fetch.php?tok=d061dd&media=https%3A%2F%2Fs3.bmp.ovh%2Fimgs%2F2024%2F10%2F09%2Fa9136fe527462fd0.png)](http://localhost/lib/exe/fetch.php?tok=d061dd&media=https%3A%2F%2Fs3.bmp.ovh%2Fimgs%2F2024%2F10%2F09%2Fa9136fe527462fd0.png)

说明:`VPC3`模拟分部，`VPC4`模拟总部，`RSR1/RSR2`均为出口路由

### 实验需求

1.配置GRE的隧道，通过ospf宣告网段建立邻居且不能宣告公网地址。

2.VPC3和VPC4能够通信，需要对数据进行加密保护。

### 配置

#### 基础配置

```
Router>enable  
Router#configure terminal 
Router(config)#hostname RSR1
RSR1(config)#int gi 0/1 
RSR1(config-if-GigabitEthernet 0/1)#ip add 192.168.1.254 24 
RSR1(config-if-GigabitEthernet 0/1)#int gi 0/0 
RSR1(config-if-GigabitEthernet 0/0)#ip add 10.1.1.2 24 
 
Router>enable  
Router#configure terminal 
Router(config)#hostname RSR
RSR(config)#int gi 0/0 
RSR(config-if-GigabitEthernet 0/0)#ip add 10.1.1.1 24 
RSR(config-if-GigabitEthernet 0/0)#int gi 0/1 
RSR(config-if-GigabitEthernet 0/1)#ip add 11.1.1.1 24 
RSR(config-if-GigabitEthernet 0/1)#ip route 
 
 
Router>enable  
Router#configure terminal 
Router(config)#hostname RSR2
RSR2(config)#int gi 0/0 
RSR2(config-if-GigabitEthernet 0/0)#ip add 11.1.1.2 24 
RSR2(config-if-GigabitEthernet 0/0)#int gi 0/1 
RSR2(config-if-GigabitEthernet 0/1)#ip add 192.168.2.254 24 
RSR2(config-router)#ip route 0.0.0.0 0.0.0.0 11.1.1.1 
```

#### GRE配置

```
RSR1(config)#int tunnel 10 
RSR1(config-if-Tunnel 10)#tunnel mode gre  ip 
RSR1(config-if-Tunnel 10)#tunnel  source 10.1.1.2 
RSR1(config-if-Tunnel 10)#tunnel destination 11.1.1.2 
RSR1(config-if-Tunnel 10)#ip address 13.1.1.1 30 
RSR1(config)#ip route  0.0.0.0 0.0.0.0 10.1.1.1
RSR1(config)#route ospf 1
RSR1(config-router)#network 13.1.1.0 0.0.0.3 area 0 
RSR1(config-router)#network  192.168.1.0 0.0.0.255  area 0 
 
 
 
RSR2(config)#int tunnel 10 
RSR2(config-if-Tunnel 10)#tunnel mode gre  ip 
RSR2(config-if-Tunnel 10)#tunnel  source  11.1.1.2 
RSR2(config-if-Tunnel 10)#tunnel destination 10.1.1.2 
RSR2(config-if-Tunnel 10)#ip add 13.1.1.2 30 
RSR2(config)#route ospf 1
RSR2(config-router)#network  13.1.1.0 0.0.0.3  area  0 
RSR2(config-router)#network 192.168.2.0 0.0.0.255 area 0 
```

### IPsec VPN配置

#### 要点

```
（1）配置ipsec感兴趣流
（2）配置isakmp策略
（3）配置预共享密钥
（4）配置ipsec加密转换集
（5）配置动态ipsec加密图映射到静态的ipsec加密图中
（6）将加密图应用到接口
 
注意：内网1和内网2需要互访的IP网段不能重叠。
```

|            | GRE over IPsec                                               | IPsec over GRE                                               |
| ---------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| ACL定义：  | 以GRE隧道源目地址为感兴趣流：access-list 101 permit gre host 222.100.100.1 host 222.200.200.1 | 以内网地址为感兴趣流：access-list 101 permit ip 192.168.1.0 0.0.0.255 192.168.2.0 0.0.0.255 |
| 应用端口： | 公网出口 ：interface GigabitEthernet 0/0cryto map mamap      | GRE Tunnel上：interface tunnel 1cryto map mamap              |

#### 配置

```
RSR1:
RSR1(config)#access-list 100 permit ip 10.1.1.2 0.0.0.0  11.1.1.2  0.0.0.0
 
RSR1(config)#crypto isakmp policy 1 
RSR1(isakmp-policy)#authentication  pre-share 
RSR1(isakmp-policy)#group  2
RSR1(isakmp-policy)#encryption 3des 
 
RSR1(config)#crypto isakmp key 0 ruijie address 11.1.1.2 
 
RSR1(config)#crypto ipsec transform-set myset esp-des esp-md5-hmac 
 
RSR1(config)#crypto map mymap 10 ipsec-isakmp 
RSR1(config-crypto-map)#set peer 11.1.1.2 
RSR1(config-crypto-map)#set transform-set myset 
RSR1(config-crypto-map)#match address 100 
 
RSR1(config)#int gi 0/0 
RSR1(config-if-GigabitEthernet 0/0)#crypto map mymap 
 
 
RSR2:
RSR2(config)#access-list 100 permit  ip 11.1.1.2 0.0.0.0 10.1.1.2 0.0.0.0 
 
RSR2(config)#crypto isakmp policy 1 
RSR2(isakmp-policy)#authentication pre-share 
RSR2(isakmp-policy)#group 2
RSR2(isakmp-policy)#encryption 3des 
 
RSR2(config)#crypto isakmp key 0 ruijie address 10.1.1.2 
 
RSR2(config)#crypto ipsec transform-set myset esp-des  esp-md5-hmac 
 
RSR2(config)#crypto map mymap 10 ipsec-isakmp  
RSR2(config-crypto-map)#set peer 10.1.1.2 
RSR2(config-crypto-map)#set transform-set myset 
RSR2(config-crypto-map)#match address 100 
RSR2(config-crypto-map)#exit 
 
RSR2(config)#int gi 0/0 
RSR2(config-if-GigabitEthernet 0/0)#crypto map mymap 
```

### 测试

1.使用VPC3访问VPC4并抓包

```
VPCS> ping 192.168.2.1
 
84 bytes from 192.168.2.1 icmp_seq=1 ttl=62 time=7.395 ms
84 bytes from 192.168.2.1 icmp_seq=2 ttl=62 time=9.661 ms
84 bytes from 192.168.2.1 icmp_seq=3 ttl=62 time=7.621 ms
84 bytes from 192.168.2.1 icmp_seq=4 ttl=62 time=6.378 ms
84 bytes from 192.168.2.1 icmp_seq=5 ttl=62 time=6.661 ms
```

[![img](http://localhost/lib/exe/fetch.php?tok=e614fa&media=https%3A%2F%2Fs3.bmp.ovh%2Fimgs%2F2024%2F10%2F09%2Faa193e67505dddfc.png)](http://localhost/lib/exe/fetch.php?tok=e614fa&media=https%3A%2F%2Fs3.bmp.ovh%2Fimgs%2F2024%2F10%2F09%2Faa193e67505dddfc.png)

2.查看isakmp sa是否已经协商成功

```
RSR1(config)#show crypto isakmp sa 
 destination       source            state                    conn-id           lifetime(second) 
 11.1.1.2          10.1.1.2          IKE_IDLE                 1                 83983    
```