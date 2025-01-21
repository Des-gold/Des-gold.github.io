## IPsec静态实验

#### 简介

> IPSec VPN分为两个协商阶段，ISAKMP阶段及IPSec阶段，ISAKMP阶段主要是协商两端的保护策略，验证对等体的合法性，产生加密密钥，保护第二阶段IPSec SA的协商。第二阶段IPSec 阶段，主要是确定IPSec SA的保护策略，使用AH还是ESP、传输模式还是隧道模式、被保护的数据是什么等等。第二阶段协商的目标就是产生真正用于保护IP数据的IPSec SA。 IPSec通信实体双方对于一、二阶段的这些安全策略必须达成一致，否则IPSec协商将无法通过。

#### 实验拓扑

[![img](http://localhost/lib/exe/fetch.php?tok=b1677f&media=https%3A%2F%2Fs3.bmp.ovh%2Fimgs%2F2024%2F10%2F07%2F30a1ba20a4f95941.png)](http://localhost/lib/exe/fetch.php?tok=b1677f&media=https%3A%2F%2Fs3.bmp.ovh%2Fimgs%2F2024%2F10%2F07%2F30a1ba20a4f95941.png)

#### 实验需求

1.两个局域网分别通过三台RSR路由器接入internet（或者专网），同时这两个局域网的网段`192.168.1.0/24`和`192.168.2.0/24`间有通信需求，需要使用`NAT`技术，并且要对通信流量进行加密。

2.该场景通过在两台RSR路由器上部署静态的`IPSEC VPN`来实现局域网间的通信及数据加密需求。

#### 配置

#### 基础配置

```
Router>enable 
Router#configure terminal 
Router(config)#hostname RSR2 
RSR2(config)#int gi 0/0 
RSR2(config-if-GigabitEthernet 0/0)#ip add 10.1.1.1 24
RSR2(config-if-GigabitEthernet 0/0)#int gi 0/1
RSR2(config-if-GigabitEthernet 0/1)#ip add 11.1.1.1 24
RSR2(config-if-GigabitEthernet 0/1)#exit 
RSR2(config)#ip route 192.168.1.0 255.255.255.0 10.1.1.2 
RSR2(config)#ip route 192.168.2.0 255.255.255.0 11.1.1.2 
 
 
Router>enable 
Router#configure terminal 
Router(config)#hostname RSR1
RSR1(config)#int gi 0/0 
RSR1(config-if-GigabitEthernet 0/0)#ip add 10.1.1.2 24 
RSR1(config-if-GigabitEthernet 0/0)#int gi 0/1 
RSR1(config-if-GigabitEthernet 0/1)#ip add 192.168.1.254 24 
 
 
Router>enable 
Router#configure terminal
Router(config)#hostname RSR3
RSR3(config)#int gi 0/0 
RSR3(config-if-GigabitEthernet 0/0)#ip add 11.1.1.2 24 
RSR3(config-if-GigabitEthernet 0/0)#int gi 0/1 
RSR3(config-if-GigabitEthernet 0/1)#ip add 192.168.2.254 24 
```

#### NAT配置

> 当接口上要同时使用NAT和IPsec时会面临一个问题，配置完后IPsec后可能不生效？
> 因为流量会优先匹配NAT流量，所以需要在NAT匹配流量中拒绝IPsec需要加密的流量

```
RSR1(config)#int gi 0/0 
RSR1(config-if-GigabitEthernet 0/0)#ip nat outside 
RSR1(config-if-GigabitEthernet 0/0)#int gi 0/1 
RSR1(config-if-GigabitEthernet 0/1)#ip nat inside 
 
RSR1(config)#ip access-list extended 100 
RSR1(config-ext-nacl)#5 deny ip 192.168.1.0 0.0.0.255 192.168.2.0 0.0.0.255 
RSR1(config-ext-nacl)#10 permit ip 192.168.1.0 0.0.0.255 any
 
RSR1(config)#ip nat inside source list 100 interface GigabitEthernet 0/0 overload
 
 
 
RSR3(config)#int gi 0/0 
RSR3(config-if-GigabitEthernet 0/0)#ip nat outside 
RSR3(config-if-GigabitEthernet 0/0)#int gi 0/1 
RSR3(config-if-GigabitEthernet 0/1)#ip nat inside 
 
RSR3(config)#ip access-list extended 100 
RSR3(config-ext-nacl)#5 deny  ip 192.168.2.0 0.0.0.255 192.168.1.0 0.0.0.255
RSR3(config-ext-nacl)#10 permit ip 192.168.2.0 0.0.0.255 any 
 
RSR3(config)#ip nat inside source list 100 interface GigabitEthernet 0/0 overload
```

#### IPsec VPN静态配置

#### 要点

```
（1）配置ipsec感兴趣流
（2）配置isakmp策略
（3）配置预共享密钥
（4）配置ipsec加密转换集
（5）配置ipsec加密图
（6）将加密图应用到接口
```

#### 配置

```
RSR1:
RSR1(config)#access-list 101 permit ip 192.168.1.0 0.0.0.255 192.168.2.0 0.0.0.255  	//配置ipsec感兴趣流
 
RSR1(config)#crypto isakmp policy  1 		//创建新的isakmp策略
RSR1(isakmp-policy)#authentication pre-share	//指定认证方式为“预共享密码” 	
RSR1(isakmp-policy)#group 2 			//设置DH组为group2
RSR1(isakmp-policy)#encryption 3des		//指定使用3DES进行加密
 
RSR1(config)#crypto  isakmp key 0 ruijie address 11.1.1.2   //指定peer 11.1.1.2的预共享密钥为“ruijie”，对端也必须配置一致的密钥。
RSR1(config)#crypto  ipsec transform-set  myset esp-des esp-md5-hmac 	  //指定ipsec使用esp封装des加密、MD5检验
 
 
RSR1(config)#crypto map mymap 5 ipsec-isakmp 	//新建名称为“mymap”的加密图    
RSR1(config-crypto-map)#set peer 11.1.1.2 	//指定peer地址   
RSR1(config-crypto-map)#set  transform-set myset 	//指定加密转换集“myset”    
RSR1(config-crypto-map)#match address 101 	//指定感兴趣流为ACL 101
 
RSR1(config)#int gi 0/0 
RSR1(config)#crypto map mymap
 
 
 
RSR3:
RSR3(config)#access-list 101 permit ip 192.168.2.0 0.0.0.255 192.168.1.0 0.0.0.255  	//配置ipsec感兴趣流
 
RSR3(config)#crypto isakmp policy  1 		//创建新的isakmp策略
RSR3(isakmp-policy)#authentication pre-share	//指定认证方式为“预共享密码” 	
RSR3(isakmp-policy)#group 2 			//设置DH组为group2
RSR3(isakmp-policy)#encryption 3des		//指定使用3DES进行加密
 
RSR3(config)#crypto  isakmp key 0 ruijie address 10.1.1.2   //指定peer 10.1.1.2的预共享密钥为“ruijie”，对端也必须配置一致的密钥。
RSR3(config)#crypto  ipsec transform-set  myset esp-des esp-md5-hmac 	  //指定ipsec使用esp封装des加密、MD5检验
 
RSR3(config)#crypto map mymap 5 ipsec-isakmp 	//新建名称为“mymap”的加密图    
RSR3(config-crypto-map)#set peer 10.1.1.2	//指定peer地址   
RSR3(config-crypto-map)#set  transform-set myset 	//指定加密转换集“myset”    
RSR3(config-crypto-map)#match address 101 	//指定感兴趣流为ACL 101
 
RSR3(config)#int gi 0/0 
RSR3(config)#crypto map mymap
```

#### 测试

#### 1.使用VPC4访问VPC5

```
VPCS> ping 192.168.2.1  
 
84 bytes from 192.168.2.1 icmp_seq=1 ttl=62 time=12.527 ms
84 bytes from 192.168.2.1 icmp_seq=2 ttl=62 time=9.784 ms
84 bytes from 192.168.2.1 icmp_seq=3 ttl=62 time=10.065 ms
84 bytes from 192.168.2.1 icmp_seq=4 ttl=62 time=9.601 ms
84 bytes from 192.168.2.1 icmp_seq=5 ttl=62 time=10.186 ms
```

#### 2.在RSR1上查看isakmp、ipsec sa是否已经协商成功

```
RSR1#show crypto isakmp sa
 destination       source            state                    conn-id           lifetime(second) 
 11.1.1.2          10.1.1.2          IKE_IDLE                 1                 81791   
```



## IPsec动态实验

#### 简介

> IPSec VPN分为两个协商阶段，ISAKMP阶段及IPSec阶段，ISAKMP阶段主要是协商两端的保护策略，验证对等体的合法性，产生加密密钥，保护第二阶段IPSec SA的协商。第二阶段IPSec 阶段，主要是确定IPSec SA的保护策略，使用AH还是ESP、传输模式还是隧道模式、被保护的数据是什么等等。第二阶段协商的目标就是产生真正用于保护IP数据的IPSec SA。 IPSec通信实体双方对于一、二阶段的这些安全策略必须达成一致，否则IPSec协商将无法通过。

#### 实验拓扑

[![img](http://localhost/lib/exe/fetch.php?tok=d9e60e&media=https%3A%2F%2Fs3.bmp.ovh%2Fimgs%2F2024%2F10%2F08%2F41c5777f789d8254.png)](http://localhost/lib/exe/fetch.php?tok=d9e60e&media=https%3A%2F%2Fs3.bmp.ovh%2Fimgs%2F2024%2F10%2F08%2F41c5777f789d8254.png)

说明:若总部的IP地址固定，而分公司是采用拨号方式上网的（IP地址不固定），那么可以采用动态IPSec VPN。我将`11.1.1.0/24`和`12.1.1.0/24`模拟为`无固定IP`

#### 实验需求

1.分支1能够访问总部的财务部门，分支2能够访问人事部，需要为数据加密。

#### 配置

#### 基础配置

```
Router>enable 
Router#configure terminal 
Router(config)#hostname  RSR3
RSR3(config)#int gi 0/0 
RSR3(config-if-GigabitEthernet 0/0)#ip 11.1.1.1 24 
RSR3(config-if-GigabitEthernet 0/0)#int gi 0/1 
RSR3(config-if-GigabitEthernet 0/1)#ip add 12.1.1.1 24 
RSR3(config-if-GigabitEthernet 0/1)#exit 
RSR3(dhcp-config)#int gi 0/2 
RSR3(config-if-GigabitEthernet 0/2)#ip add 10.1.1.1 24 
RSR3(config)#ip route 172.16.1.0 255.255.255.0 10.1.1.2 
RSR3(config)#ip route 172.16.2.0 255.255.255.0 10.1.1.2 
RSR3(config)#ip route  192.168.1.0 255.255.255.0 11.1.1.2 
RSR3(config)#ip route  192.168.2.0 255.255.255.0 12.1.1.2 
 
 
 
Router>enable 
Router#configure terminal 
Route(config)#hostname RSR4 
RSR4(config)#int gi 0/0 
RSR4(config-if-GigabitEthernet 0/0)#ip add 10.1.1.2 24 
RSR4(config-if-GigabitEthernet 0/0)#ip route 0.0.0.0 0.0.0.0 10.1.1.1
RSR4(config)#int loop 0 
RSR4(config-if-Loopback 0)#ip add 172.16.1.1 24 
RSR4(config-if-Loopback 0)#int loop 1
SR4(config-if-Loopback 1)#ip add 172.16.2.1 24
 
 
Router>enable 
Router#configure terminal
Router(config)#hostname RSR1
RSR1(config)#int gi 0/0 
RSR1(config-if-GigabitEthernet 0/0)#ip add 11.1.1.2 24 
RSR1(config-if-GigabitEthernet 0/0)#int gi 0/1 
RSR1(config-if-GigabitEthernet 0/1)#ip add 192.168.1.254 24 
RSR1(config)#ip route 172.16.1.0 255.255.255.0 10.1.1.2 
RSR1(config)#ip route 0.0.0.0 0.0.0.0 11.1.1.1 
 
 
Router>enable 
Router#configure terminal
Router(config)#hostname RSR2
RSR2(config)#int gi 0/0 
RSR2(config-if-GigabitEthernet 0/0)#ip add 12.1.1.2 24 
RSR2(config-if-GigabitEthernet 0/0)#int gi 0/1 
RSR2(config-if-GigabitEthernet 0/1)#ip add 192.168.2.254 24 
RSR2(config-if-GigabitEthernet 0/1)#ip route 0.0.0.0 0.0.0.0 12.1.1.1
```

#### NAT配置

> 当接口上要同时使用NAT和IPsec时会面临一个问题，配置完后IPsec后可能不生效？
> 因为流量会优先匹配NAT流量，所以需要在NAT匹配流量中拒绝IPsec需要加密的流量

```
RSR1(config)#int gi 0/0 
RSR1(config-if-GigabitEthernet 0/0)#ip nat outside 
RSR1(config-if-GigabitEthernet 0/0)#int gi 0/1 
RSR1(config-if-GigabitEthernet 0/1)#ip nat inside 
 
RSR1(config)#ip access-list extended 101 
RSR1(config-ext-nacl)#5 deny ip 192.168.1.0 0.0.0.255 172.16.1.0 0.0.0.255 
RSR1(config-ext-nacl)#10 permit ip 192.168.1.0  0.0.0.255 any 
RSR1(config-ext-nacl)#exit 
 
RSR1(config)#ip nat inside source list 101 interface GigabitEthernet 0/0 overload
 
 
 
RSR3(config)#int gi 0/0 
RSR3(config-if-GigabitEthernet 0/0)#ip nat outside 
RSR3(config-if-GigabitEthernet 0/0)#int gi 0/1 
RSR3(config-if-GigabitEthernet 0/1)#ip nat inside 
 
RSR3(config)#ip access-list extended 101 
RSR3(config-ext-nacl)#5 deny ip 192.168.2.0 0.0.0.255 172.16.2.0 0.0.0.255 
RSR3(config-ext-nacl)#10 permit ip 192.168.2.0  0.0.0.255 any 
RSR3(config-ext-nacl)#exit 
 
RSR3(config)#ip nat inside source list 101 interface GigabitEthernet 0/0 overload
```

#### IPsec VPN配置

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

#### 配置

#### 总部出口路由

```
RSR3:
RSR3(config)#crypto isakmp policy 1 		//创建新的isakmp策略 
RSR3(isakmp-policy)#authentication pre-share 	//指定认证方式为“预共享密码”
RSR3(isakmp-policy)#encryption 3des		//指定使用3DES进行加密 
 
RSR3(config)#crypto isakmp key 0 ruijie address 0.0.0.0 0.0.0.0    //配置预共享密钥为“ruijie”，由于对端的 ip地址是动态的，因此使用address 0.0.0.0 0.0.0.0代表所有ipsec客户端 
 
RSR3(config)#crypto ipsec transform-set myset esp-des esp-md5-hmac 	//指定ipsec使用esp封装des加密、MD5检验 
RSR3(config)#crypto dynamic-map mymap 5 	//新建名为“mymap”的动态ipsec加密图 
RSR3(config-crypto-map)#set transform-set myset 	//指定加密转换集为“myset” 
 
RSR3(config)#crypto  map Smap 10 ipsec-isakmp dynamic mymap 	//将动态的“mymap”ipsec加密图映射至静态ipsec加密图Smap中
 
RSR3(config)#int gi 0/0 
RSR3(config-if-GigabitEthernet 0/0)#crypto map Smap
```

#### 以分支1为例

分支2的配置类似（略）

```
RSR1(config)#access-list 101 permit ip 192.168.1.0 0.0.0.255 172.16.1.0 0.0.0.255 
  
RSR1(config)#crypto isakmp policy 1 
RSR1(isakmp-policy)#authentication pre-share 
RSR1(isakmp-policy)#encrypti 3des 
RSR1(isakmp-policy)#exit 

RSR1(config)#crypto isakmp key 0 ruijie address 11.1.1.1 255.255.255.0 

RSR1(config)#crypto ipsec transform-set myset esp-des esp-md5-hmac 

RSR1(config)#crypto map mymap 5 ipsec-isakmp 
RSR1(config-crypto-map)#set peer 11.1.1.1
RSR1(config-crypto-map)#match address 100  

RSR1(config-crypto-map)#int gi 0/0 
RSR1(config-if-GigabitEthernet 0/0)#crypto map mymap 
```

#### 测试

#### 1.使用VPC5访问财务部

```
VPCS> ping 172.16.1.1  
 
84 bytes from 172.16.1.1 icmp_seq=1 ttl=62 time=7.082 ms
84 bytes from 172.16.1.1 icmp_seq=2 ttl=62 time=5.280 ms
84 bytes from 172.16.1.1 icmp_seq=3 ttl=62 time=6.031 ms
84 bytes from 172.16.1.1 icmp_seq=4 ttl=62 time=9.700 ms
84 bytes from 172.16.1.1 icmp_seq=5 ttl=62 time=8.819 ms
```









## IPsec野蛮模式实验

#### 简介

> IPSEC动态隧道一般应用于分支节点较多的总分拓扑结构，在中心点配置动态隧道接受各分支的IPSEC VPN拨入。中心点配置简单、易于维护、可扩展性强。同时由于各分支的IP地址不固定，无法通过IP地址来指定不同的预共享密钥。所有分支使用相同的预共享密钥将极易造成密钥泄露，威胁网络安全。使用域名认证的方法很好地解决了这个问题，通过每个分支配置使用不同的域名，同时在中心点上针对不同的域名指定不同的密钥，保证了密钥安全。

#### 实验拓扑

[![img](http://localhost/lib/exe/fetch.php?tok=6be69a&media=https%3A%2F%2Fs3.bmp.ovh%2Fimgs%2F2024%2F10%2F09%2Fcaac8eaa30b8d1a7.png)](http://localhost/lib/exe/fetch.php?tok=6be69a&media=https%3A%2F%2Fs3.bmp.ovh%2Fimgs%2F2024%2F10%2F09%2Fcaac8eaa30b8d1a7.png)

说明:`RSR1`的`G0/0`无法模拟无固定地址，故以`10.1.1.2`模拟。

#### 实验需求

1.分支1能够访问总部，需要保护数据使用IPsec加密。

2.RSR1无固定公网IP使用fqdn定义设备。

#### 配置

#### 基础配置

```
Router>enable 
Router#configure terminal 
Router(config)#hostname RSR1
RSR1(config)#int loop 0 
RSR1(config-if-Loopback 0)#ip add 192.168.1.1 24 
RSR1(config-if-Loopback 0)#int gi 0/0 
RSR1(config-if-GigabitEthernet 0/0)#ip add 10.1.1.2 24 
RSR1(config)#ip route 0.0.0.0 0.0.0.0 10.1.1.1
 
 
Router>enable 
Router#configure terminal 
Router(config)#hostname RSR2
RSR2(config)#int gi 0/0 
RSR2(config-if-GigabitEthernet 0/0)#ip add 10.1.1.1 24 
RSR2(config-if-GigabitEthernet 0/0)#int gi 0/1 
RSR2(config-if-GigabitEthernet 0/1)#ip add 11.1.1.1 24 
RSR2(config)#ip route 192.168.1.0 255.255.255.0 10.1.1.2
RSR2(config)#ip route 172.168.1.0 255.255.255.0 11.1.1.2
 
 
Router>enable 
Router#configure terminal 
Router(config)#hostname RSR3
RSR3(config)#int loop 0 
RSR3(config-if-Loopback 0)#ip add 172.168.1.1 24 
RSR3(config-if-Loopback 0)#int gi 0/0 
RSR3(config-if-GigabitEthernet 0/0)#ip add 11.1.1.2 24 
RSR3(config)#ip route 0.0.0.0 0.0.0.0 11.1.1.1 
```

#### IPsec VPN静态配置

#### 要点

```
（1）配置isakmp策略     
（2）配置预共享密钥     
（3）配置isakmp模式自动识别
（3）配置ipsec加密转换集     
（4）配置动态ipsec加密图     
（5）将动态ipsec加密图映射到静态的ipsec加密图中    
（6）将加密图应用到接口  
```

#### 配置

```
RSR3:
RSR3(config)#crypto  isakmp policy  1 
RSR3(isakmp-policy)#encryption 3des 
RSR3(isakmp-policy)#authentication pre-share 
 
RSR3(config)#crypto isakmp key 0 ruijie hostname test1.kim6.cn   //分别配置各个分支的预共享密钥，并通过hostname指定各分支的名称。
 
RSR3(config)#crypto isakmp mode-detect 		//配置isakmp模式自动识别，这样才能够接受来自分支的IKE积极模式的协商    
 
RSR3(config)#crypto ipsec transform-set myset  esp-des  esp-md5-hmac 
 
RSR3(config)#crypto dynamic-map dymap 5 
RSR3(config-crypto-map)#set transform-set myset
 
RSR3(config)#crypto map mymap 10 ipsec-isakmp dynamic dymap   //将动态的“dymymap”ipsec加密图映射至静态ipsec加密图mymap中
 
 
RSR3(config)#int gi 0/0 
RSR3(config-if-GigabitEthernet 0/0)#crypto map mymap
 
 
RSR1:
RSR1(config)#self-identity fqdn test1.kim6.cn
RSR1(config)#access-list 101 permit ip  192.168.1.0 0.0.0.255 172.168.1.0 0.0.0.255 
 
RSR1(config)#crypto isakmp policy 1 
RSR1(isakmp-policy)#encryption 3des 
RSR1(isakmp-policy)#authentication pre-share
 
RSR1(config)#crypto isakmp key 0 ruijie address 11.1.1.2 
 
RSR1(config)#crypto ipsec transform-set myset esp-des esp-md5-hmac 
 
RSR1(config)#crypto map mymap 5  ipsec-isakmp 
RSR1(config-crypto-map)#set peer 11.1.1.2  
RSR1(config-crypto-map)#set transform-set myset 
RSR1(config-crypto-map)#set exchange-mode aggressive   //指定采用积极模式发起IKE协商
RSR1(config-crypto-map)#match  address 101 
 
 
RSR1(config-crypto-map)#int gi 0/0 
RSR1(config-if-GigabitEthernet 0/0)#crypto map mymap
```

#### 测试

#### 1.使用VPC4访问VPC5

```
RSR1#ping 172.168.1.1 source 192.168.1.1
Sending 5, 100-byte ICMP Echoes to 172.168.1.1, timeout is 2 seconds:
  < press Ctrl+C to break >
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 4/4/7 ms.
```

#### 2.在RSR1上查看isakmp sa是否已经协商成功

```
RSR1#show crypto isakmp sa
 destination       source            state                    conn-id           lifetime(second) 
 11.1.1.2          10.1.1.2          IKE_IDLE                 1       
```