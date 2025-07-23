## 跨域VPN-OptionA配置

#### 简介

> 跨域VPN-OptionA的实现比较简单，当PE上的VPN数量及VPN路由数量都比较少时可以采用这种方案。
>
> 跨域VPN-OptionA的配置可以描述为：
>
> - 对各AS分别进行基本MPLS L3VPN配置。
>
> - 对于ASBR，将对端ASBR看作自己的CE配置即可。即：跨域VPN-OptionA方式需要在PE和ASBR上分别配置VPN实例，前者用于接入CE，后者用于接入对端ASBR。
>
> 在跨域VPN-OptionA方式中，对于同一个VPN，同一AS内的ASBR与PE的VPN实例的Route Target应能匹配；不同AS的PE之间的VPN实例的Route Target则不需要匹配。

### 1.组网图

![](https://kim-1259075102.cos.ap-guangzhou.myqcloud.com/HCL-Picture/optiona_v4.png)

| 设备     | 接口      | IP地址       | 设备     | 接口      | IP地址       |
| -------- | --------- | ------------ | -------- | --------- | ------------ |
| CE1      | GE0/1     | 10.1.1.1/24  | CE2      | GE0/1     | 10.2.1.1/24  |
| PE1      | GE0/0     | 172.1.1.2/24 | PE2      | GE0/0     | 162.1.1.2/24 |
|          | GE0/1     | 10.1.1.2/24  |          | GE0/1     | 10.2.1.2/24  |
|          | Loopback0 | 1.1.1.1/32   |          | Loopback0 | 4.4.4.4/32   |
| ASBR-PE1 | GE0/0     | 172.1.1.1/24 | ASBR-PE2 | GE0/0     | 162.1.1.1/24 |
|          | GE0/1     | 192.1.1.1/24 |          | GE0/1     | 192.1.1.2/24 |
|          | Loopback0 | 2.2.2.2/32   |          | Loopback0 | 3.3.3.3/32   |

### 2.组网需求

- CE 1和CE 2属于同一个VPN。
- CE 1通过AS 100的PE 1接入，CE 2通过AS 200的PE 2接入。
- 采用OptionA方式实现跨域的MPLS L3VPN，即，采用VRF-to-VRF方式管理VPN路由。
- 同一个AS内部的MPLS骨干网使用OSPF作为IGP。

### 3.配置步骤

```bash
(1)配置各接口的IP地址（略）
(2)在MPLS骨干网上配置IGP协议，实现骨干网内互通
#配置PE1
[PE1]ospf 1
[PE1-ospf-1] area 0.0.0.0
[PE1-ospf-1-area-0.0.0.0]  network 1.1.1.1 0.0.0.0
[PE1-ospf-1-area-0.0.0.0]  network 172.1.1.0 0.0.0.255

#配置ASBR-PE1
[ASBR-PE1]ospf 1
[ASBR-PE1-ospf-1] area 0.0.0.0
[ASBR-PE1-ospf-1-area-0.0.0.0]  network 2.2.2.2 0.0.0.0
[ASBR-PE1-ospf-1-area-0.0.0.0]  network 172.1.1.0 0.0.0.255

#配置PE2
[PE2]ospf 1
[PE2-ospf-1] area 0.0.0.0
[PE2-ospf-1-area-0.0.0.0]  network 4.4.4.4 0.0.0.0
[PE2-ospf-1-area-0.0.0.0]  network 162.1.1.0 0.0.0.255

#配置ASBR-PE2
[ASBR-PE2]ospf 1 
[ASBR-PE2-ospf-1] area 0.0.0.0
[ASBR-PE2-ospf-1-area-0.0.0.0]  network 3.3.3.3 0.0.0.0
[ASBR-PE2-ospf-1-area-0.0.0.0]  network 162.1.1.0 0.0.0.255

(3)在MPLS骨干网上配置MPLS基本能力和MPLS LDP，建立LDP LSP
#配置PE1的MPLS基本能力，并在与ASBR-PE1相连的接口上使能LDP。
[PE1]mpls lsr-id 1.1.1.1
[PE1]mpls ldp
[PE1-ldp]quit
[PE1]interface GigabitEthernet0/0
[PE1-GigabitEthernet0/0] mpls enable
[PE1-GigabitEthernet0/0] mpls ldp enable
[PE1-GigabitEthernet0/0]quit

#配置ASBR-PE1的MPLS基本能力，并在与PE1相连的接口上使能LDP。
[ASBR-PE1]mpls lsr-id 2.2.2.2
[ASBR-PE1]mpls ldp
[ASBR-PE1-ldp]quit
[ASBR-PE1]interface GigabitEthernet0/0
[ASBR-PE1-GigabitEthernet0/0] mpls enable
[ASBR-PE1-GigabitEthernet0/0] mpls ldp enable
[ASBR-PE1-GigabitEthernet0/0]quit

#配置PE2的MPLS基本能力，并在与ASBR-PE2相连的接口上使能LDP
[PE2]mpls lsr-id 4.4.4.4
[PE2]mpls ldp
[PE2-ldp]quit
[PE2]interface GigabitEthernet0/0
[PE2-GigabitEthernet0/0] mpls enable
[PE2-GigabitEthernet0/0] mpls ldp enable
[PE2-GigabitEthernet0/0]quit

#配置ASBR-PE2的MPLS基本能力，并在与PE2相连的接口上使能LDP
[ASBR-PE2]mpls lsr-id 3.3.3.3
[ASBR-PE2]mpls ldp
[ASBR-PE2-ldp]quit
[ASBR-PE2]interface GigabitEthernet0/0
[ASBR-PE2-GigabitEthernet0/0] mpls enable
[ASBR-PE2-GigabitEthernet0/0] mpls ldp enable
[ASBR-PE2-GigabitEthernet0/0]quit
上述配置完成后，同一AS的PE和ASBR-PE之间应该建立起LDP邻居，在各设备上执行
display mpls ldp peer命令可以看到LDP会话状态为“Operational”。

(4)在PE设备上配置VPN实例，将CE接入PE
#配置PE1
[PE1]ip vpn-instance vpn1
[PE1-vpn-instance-vpn1] route-distinguisher 100:2
[PE1-vpn-instance-vpn1] vpn-target 100:1 both
[PE1]interface GigabitEthernet0/1
[PE1-GigabitEthernet0/1] ip binding vpn-instance vpn1
[PE1-GigabitEthernet0/1] ip address 10.1.1.2 255.255.255.0

#配置PE2
[PE2]ip vpn-instance vpn1
[PE2-vpn-instance-vpn1] route-distinguisher 200:2
[PE2-vpn-instance-vpn1] vpn-target 200:1 both
[PE2]interface GigabitEthernet0/1
[PE2-GigabitEthernet0/1] ip binding vpn-instance vpn1
[PE2-GigabitEthernet0/1] ip address 10.2.1.2 255.255.255.0

#配置ASBR-PE1：创建VPN实例，并将此实例绑定到连接ASBR-PE2的接口（ASBR-PE1认为ASBR-PE2是自己的CE）
[ASBR-PE1]ip vpn-instance vpn1
[ASBR-PE1-vpn-instance-vpn1] route-distinguisher 100:1
[ASBR-PE1-vpn-instance-vpn1] vpn-target 100:1 both
[ASBR-PE1]interface GigabitEthernet0/1
[ASBR-PE1-GigabitEthernet0/1] ip binding vpn-instance vpn1
[ASBR-PE1-GigabitEthernet0/1] ip address 192.1.1.1 255.255.255.0

# 配置ASBR-PE2：创建VPN实例，并将此实例绑定到连接ASBR-PE1的接口（ASBR-PE2认为ASBR-PE1是自己的CE）
[ASBR-PE2]ip vpn-instance vpn1
[ASBR-PE2-vpn-instance-vpn1] route-distinguisher 200:1
[ASBR-PE2-vpn-instance-vpn1] vpn-target 200:1 both
[ASBR-PE2]interface GigabitEthernet0/1
[ASBR-PE2-GigabitEthernet0/1] ip binding vpn-instance vpn1
[ASBR-PE2-GigabitEthernet0/1] ip address 192.1.1.2 255.255.255.0
上述配置完成后，在各PE设备上执行display ip vpn-instance命令能正确显示VPN实例配置。
各PE能ping通各自的CE。ASBR-PE之间也能互相ping通。
例：
1)PE1 ping CE1
[PE1]ping -vpn-instance  vpn1  10.1.1.1
Ping 10.1.1.1 (10.1.1.1): 56 data bytes, press CTRL+C to break
56 bytes from 10.1.1.1: icmp_seq=0 ttl=255 time=1.000 ms
...

2)ASBR-PE1 ping ASBR-PE2
[ASBR-PE1]ping -vpn-instance vpn1  192.1.1.2
Ping 192.1.1.2 (192.1.1.2): 56 data bytes, press CTRL+C to break
56 bytes from 192.1.1.2: icmp_seq=0 ttl=255 time=1.000 ms
...

(5)在PE与CE之间建立EBGP对等体，引入VPN路由
#配置CE1
[CE1]bgp 65001
[CE1-bgp-default] peer 10.1.1.2 as-number 100
[CE1-bgp-default] address-family ipv4 unicast
[CE1-bgp-default-ipv4]  import-route direct
[CE1-bgp-default-ipv4]  peer 10.1.1.2 enable

#配置PE1
[PE1]bgp 100
[PE1-bgp-default-vpnv4] ip vpn-instance vpn1
[PE1-bgp-default-vpn1]  peer 10.1.1.1 as-number 65001
[PE1-bgp-default-vpn1]  address-family ipv4 unicast
[PE1-bgp-default-ipv4-vpn1]   import-route direct
[PE1-bgp-default-ipv4-vpn1]   peer 10.1.1.1 enable

#配置CE2
[CE2]bgp 65002
[CE2-bgp-default] peer 10.2.1.2 as-number 200
[CE2-bgp-default] address-family ipv4 unicast
[CE2-bgp-default-ipv4]  import-route direct
[CE2-bgp-default-ipv4]  peer 10.2.1.2 enable

#配置PE2
[PE2]bgp 200
[PE2-bgp-default-vpnv4] ip vpn-instance vpn1
[PE2-bgp-default-vpn1]  peer 10.2.1.1 as-number 65002
[PE2-bgp-default-vpn1]  address-family ipv4 unicast
[PE2-bgp-default-ipv4-vpn1]   import-route direct
[PE2-bgp-default-ipv4-vpn1]   peer 10.2.1.1 enable
上述配置为节省时间故引入直连地址，可以用network代替


(6)PE与本AS的ASBR-PE之间建立MP-IBGP对等体，ASBR-PE之间建立EBGP对等体
#配置PE1
[PE1]bgp 100
[PE1-bgp-default] peer 2.2.2.2 as-number 100
[PE1-bgp-default] peer 2.2.2.2 connect-interface LoopBack0
[PE1-bgp-default] address-family vpnv4
[PE1-bgp-default-vpnv4]  peer 2.2.2.2 enable
[PE1-bgp-default-vpnv4]  peer 2.2.2.2 next-hop-local

#配置ASBR-PE1
[ASBR-PE1]bgp 100
[ASBR-PE1-bgp-default] peer 1.1.1.1 as-number 100
[ASBR-PE1-bgp-default] peer 1.1.1.1 connect-interface LoopBack0
[ASBR-PE1-bgp-default] address-family ipv4 unicast
[ASBR-PE1-bgp-default-ipv4] address-family vpnv4
[ASBR-PE1-bgp-default-vpnv4]  peer 1.1.1.1 enable
[ASBR-PE1-bgp-default-vpnv4]  peer 1.1.1.1 next-hop-local
[ASBR-PE1-bgp-default-vpnv4] ip vpn-instance vpn1
[ASBR-PE1-bgp-default-vpn1]  peer 192.1.1.2 as-number 200
[ASBR-PE1-bgp-default-vpn1]  address-family ipv4 unicast
[ASBR-PE1-bgp-default-ipv4-vpn1]   peer 192.1.1.2 enable

#配置PE2
[PE2]bgp 200
[PE2-bgp-default] peer 3.3.3.3 as-number 200
[PE2-bgp-default] peer 3.3.3.3 connect-interface LoopBack0
[PE2-bgp-default] address-family vpnv4
[PE2-bgp-default-vpnv4]  peer 3.3.3.3 enable
[PE2-bgp-default-vpnv4]  peer 3.3.3.3 next-hop-local

#配置ASBR-PE2
[ASBR-PE2]bgp 200
[ASBR-PE2-bgp-default] peer 4.4.4.4 as-number 200
[ASBR-PE2-bgp-default] peer 4.4.4.4 connect-interface LoopBack0
[ASBR-PE2-bgp-default] address-family vpnv4
[ASBR-PE2-bgp-default-vpnv4]  peer 4.4.4.4 enable
[ASBR-PE2-bgp-default-vpnv4]  peer 4.4.4.4 next-hop-local
[ASBR-PE2-bgp-default-vpnv4] ip vpn-instance vpn1
[ASBR-PE2-bgp-default-vpn1]  peer 192.1.1.1 as-number 100
[ASBR-PE2-bgp-default-vpn1]  address-family ipv4 unicast
[ASBR-PE2-bgp-default-ipv4-vpn1]   peer 192.1.1.1 enable
```

### 4.验证配置

```bash
上述配置完成后，CE之间能学习到对方的接口路由，CE 1和CE 2能够相互ping通。
#查看ASBR-PE1的VPNv4路由
[ASBR-PE1]display  bgp routing-table vpnv4
...

 Route distinguisher: 100:1(vpn1)
 Total number of routes: 2

     Network            NextHop         MED        LocPrf     PrefVal Path/Ogn

* >i 10.1.1.0/24        1.1.1.1         0          100        0       ?
* >e 10.2.1.0/24        192.1.1.2                             0       200?

 Route distinguisher: 100:2
 Total number of routes: 1

     Network            NextHop         MED        LocPrf     PrefVal Path/Ogn

* >i 10.1.1.0/24        1.1.1.1         0          100        0       ?

#查看ASBR-PE2的VPNv4路由
[ASBR-PE2]display  bgp routing-table vpnv4
...

 Route distinguisher: 200:1(vpn1)
 Total number of routes: 2

     Network            NextHop         MED        LocPrf     PrefVal Path/Ogn

* >e 10.1.1.0/24        192.1.1.1                             0       100?
* >i 10.2.1.0/24        4.4.4.4         0          100        0       ?

 Route distinguisher: 200:2
 Total number of routes: 1

     Network            NextHop         MED        LocPrf     PrefVal Path/Ogn

* >i 10.2.1.0/24        4.4.4.4         0          100        0       ?

#查看CE1的BGP路由表
[CE1]display  bgp routing-table ipv4
...

     Network            NextHop         MED        LocPrf     PrefVal Path/Ogn

* >  10.1.1.0/24        10.1.1.1        0                     32768   ?
*  e                    10.1.1.2        0                     0       100?
* >  10.1.1.1/32        127.0.0.1       0                     32768   ?
* >e 10.2.1.0/24        10.1.1.2                              0       100 200?

#查看CE2的BGP路由表
[CE2]display  bgp routing-table ipv4
...

     Network            NextHop         MED        LocPrf     PrefVal Path/Ogn

* >e 10.1.1.0/24        10.2.1.2                              0       200 100?
* >  10.2.1.0/24        10.2.1.1        0                     32768   ?
*  e                    10.2.1.2        0                     0       200?
* >  10.2.1.1/32        127.0.0.1       0                     32768   ?

#查看CE1 ping CE2
[CE1]ping 10.2.1.1
Ping 10.2.1.1 (10.2.1.1): 56 data bytes, press CTRL+C to break
56 bytes from 10.2.1.1: icmp_seq=0 ttl=251 time=1.000 ms
...

```





## 跨域VPN-OptionB配置

#### 简介

> ASBR在将VPNv4路由发布给MP-IBGP对等体时，始终会将下一跳修改为自身的地址，不受peer  next-hop-local命令的控制。
>
> 配置基本MPLS L3VPN，并指定同一AS内的ASBR为MP-IBGP对等体。对于同一个VPN，不同AS的PE上为该VPN实例配置的Route Target需要匹配。

### 1.组网图

![](https://kim-1259075102.cos.ap-guangzhou.myqcloud.com/HCL-Picture/optionb_v4.png)

| 设备     | 接口      | IP地址       | 设备     | 接口      | IP地址       |
| -------- | --------- | ------------ | -------- | --------- | ------------ |
| CE1      | GE0/1     | 10.1.1.1/24  | CE2      | GE0/1     | 10.2.1.1/24  |
| PE1      | GE0/0     | 172.1.1.2/24 | PE2      | GE0/0     | 162.1.1.2/24 |
|          | GE0/1     | 10.1.1.2/24  |          | GE0/1     | 10.2.1.2/24  |
|          | Loopback0 | 1.1.1.1/32   |          | Loopback0 | 4.4.4.4/32   |
| ASBR-PE1 | GE0/0     | 172.1.1.1/24 | ASBR-PE2 | GE0/0     | 162.1.1.1/24 |
|          | GE0/1     | 192.1.1.1/24 |          | GE0/1     | 192.1.1.2/24 |
|          | Loopback0 | 2.2.2.2/32   |          | Loopback0 | 3.3.3.3/32   |

### 2.组网需求

- Site 1和Site 2属于同一个VPN，Site 1的CE 1通过AS 100的PE 1接入，Site 2的CE 2通过AS 600的PE 2接入；
- 同一自治系统内的PE设备之间运行IS-IS作为IGP；
- PE 1与ASBR-PE 1间通过MP-IBGP交换VPNv4路由；
- PE 2与ASBR-PE 2间通过MP-IBGP交换VPNv4路由；
- ASBR-PE 1与ASBR-PE 2间通过MP-EBGP交换VPNv4路由；
- ASBR上不对接收的VPNv4路由进行Route  Target过滤。

### 3.配置步骤

```bash
(1)配置各接口的IP地址（略）
(2)配置MPLS骨干网上配置IGP协议(OSPF)，实现骨干网内互通(略)
(3)配置LSR ID，使能MPLS和LDP
#配置PE1
[PE1]mpls lsr-id 1.1.1.1
[PE1]mpls ldp
[PE1-ldp]quit
[PE1]interface GigabitEthernet0/0
[PE1-GigabitEthernet0/0] mpls enable
[PE1-GigabitEthernet0/0] mpls ldp enable

#配置ASBR-PE1
[ASBR-PE1]mpls lsr-id 2.2.2.2
[ASBR-PE1]mpls ldp
[ASBR-PE1-ldp]quit
[ASBR-PE1]interface GigabitEthernet0/0
[ASBR-PE1-GigabitEthernet0/0] mpls enable
[ASBR-PE1-GigabitEthernet0/0] mpls ldp enable
[ASBR-PE1]interface GigabitEthernet0/1
[ASBR-PE1-GigabitEthernet0/1] mpls enable

#配置PE2
[PE2]mpls lsr-id 4.4.4.4
[PE2]mpls ldp
[PE2-ldp]quit
[PE2]interface GigabitEthernet0/0
[PE2-GigabitEthernet0/0] mpls enable
[PE2-GigabitEthernet0/0] mpls ldp enable

#配置ASBR-PE2
[ASBR-PE2]mpls lsr-id 3.3.3.3
[ASBR-PE2]mpls ldp
[ASBR-PE2-ldp]quit
[ASBR-PE2]interface GigabitEthernet0/0
[ASBR-PE2-GigabitEthernet0/0] mpls enable
[ASBR-PE2-GigabitEthernet0/0] mpls ldp enable
[ASBR-PE2]interface GigabitEthernet0/1
[ASBR-PE2-GigabitEthernet0/1] mpls enable

(4)配置VPN实例
#配置PE1
[PE1]ip vpn-instance vpn1
[PE1-vpn-instance-vpn1] route-distinguisher 11:11
[PE1-vpn-instance-vpn1] vpn-target 1:1 2:2 import-extcommunity
[PE1-vpn-instance-vpn1] vpn-target 1:1 export-extcommunity
[PE1]interface GigabitEthernet0/1
[PE1-GigabitEthernet0/1] ip binding vpn-instance vpn1
[PE1-GigabitEthernet0/1] ip address 10.1.1.2 255.255.255.0

#配置PE2
[PE2]ip vpn-instance vpn1
[PE2-vpn-instance-vpn1] route-distinguisher 12:12
[PE2-vpn-instance-vpn1] vpn-target 1:1 2:2 import-extcommunity
[PE2-vpn-instance-vpn1] vpn-target 2:2 export-extcommunity
[PE2]interface GigabitEthernet0/1
[PE2-GigabitEthernet0/1] ip binding vpn-instance vpn1
[PE2-GigabitEthernet0/1] ip address 10.2.1.2 255.255.255.0

(5)配置MP-BGP
#配置PE1
[PE1]bgp 100
[PE1-bgp-default] peer 2.2.2.2 as-number 100
[PE1-bgp-default] peer 2.2.2.2 connect-interface LoopBack0
[PE1-bgp-default] address-family vpnv4
[PE1-bgp-default-vpnv4]  peer 2.2.2.2 enable
[PE1-bgp-default-vpnv4] ip vpn-instance vpn1
[PE1-bgp-default-vpn1]  peer 10.1.1.1 as-number 65001
[PE1-bgp-default-vpn1]  address-family ipv4 unicast
[PE1-bgp-default-ipv4-vpn1]   import-route direct
[PE1-bgp-default-ipv4-vpn1]   peer 10.1.1.1 enable

#配置PE2
[PE2]bgp 600
[PE2-bgp-default] peer 3.3.3.3 as-number 600
[PE2-bgp-default] peer 3.3.3.3 connect-interface LoopBack0
[PE2-bgp-default] address-family vpnv4
[PE2-bgp-default-vpnv4]  peer 3.3.3.3 enable
[PE2-bgp-default-vpnv4] ip vpn-instance vpn1
[PE2-bgp-default-vpn1]  peer 10.2.1.1 as-number 65002
[PE2-bgp-default-vpn1]  address-family ipv4 unicast
[PE2-bgp-default-ipv4-vpn1]   import-route direct
[PE2-bgp-default-ipv4-vpn1]   peer 10.2.1.1 enable

#配置ASBR-PE1
[ASBR-PE1]bgp 100
[ASBR-PE1-bgp-default] peer 1.1.1.1 as-number 100
[ASBR-PE1-bgp-default] peer 1.1.1.1 connect-interface LoopBack0
[ASBR-PE1-bgp-default] peer 192.1.1.2 as-number 600
[ASBR-PE1-bgp-default] peer 192.1.1.2 connect-interface GigabitEthernet0/1
[ASBR-PE1-bgp-default] address-family vpnv4
[ASBR-PE1-bgp-default-vpnv4]  peer 1.1.1.1 enable
[ASBR-PE1-bgp-default-vpnv4]  peer 192.1.1.2 enable
#不对接收的VPNv4路由进行Route target过滤
[ASBR-PE1-bgp-default-vpnv4]  undo policy vpn-target


#配置ASBR-PE2
[ASBR-PE2]bgp 600
[ASBR-PE2-bgp-default] peer 4.4.4.4 as-number 600
[ASBR-PE2-bgp-default] peer 4.4.4.4 connect-interface LoopBack0
[ASBR-PE2-bgp-default] peer 192.1.1.1 as-number 100
[ASBR-PE2-bgp-default] peer 192.1.1.1 connect-interface GigabitEthernet0/1
[ASBR-PE2-bgp-default] address-family vpnv4
[ASBR-PE2-bgp-default-vpnv4]  peer 4.4.4.4 enable
[ASBR-PE2-bgp-default-vpnv4]  peer 192.1.1.1 enable
#不对接收的VPNv4路由进行Route target过滤
[ASBR-PE2-bgp-default-vpnv4]  undo policy vpn-target

#配置CE1
[CE1]bgp 65001
[CE1-bgp-default] peer 10.1.1.2 as-number 100
[CE1-bgp-default] #
[CE1-bgp-default] address-family ipv4 unicast
[CE1-bgp-default-ipv4]  import-route direct
[CE1-bgp-default-ipv4]  peer 10.1.1.2 enable

#配置CE2
[CE2]bgp 65002
[CE2-bgp-default] peer 10.2.1.2 as-number 600
[CE2-bgp-default] address-family ipv4 unicast
[CE2-bgp-default-ipv4]  import-route direct
[CE2-bgp-default-ipv4]  peer 10.2.1.2 enable


```

### 4.验证配置

```bash
#查看PE1的BGP路由
[PE1]display  bgp routing-table vpnv4
...

     Network            NextHop         MED        LocPrf     PrefVal Path/Ogn

* >  10.1.1.0/24        10.1.1.2        0                     32768   ?
*  e                    10.1.1.1        0                     0       65001?
* >  10.1.1.2/32        127.0.0.1       0                     32768   ?
* >i 10.2.1.0/24        2.2.2.2                    100        0       600?

 Route distinguisher: 12:12
 Total number of routes: 1

     Network            NextHop         MED        LocPrf     PrefVal Path/Ogn

* >i 10.2.1.0/24        2.2.2.2                    100        0       600?

#查看PE2的BGP路由
[PE2]display  bgp routing-table vpnv4
...

 Route distinguisher: 11:11
 Total number of routes: 1

     Network            NextHop         MED        LocPrf     PrefVal Path/Ogn

* >i 10.1.1.0/24        3.3.3.3                    100        0       100?

 Route distinguisher: 12:12(vpn1)
 Total number of routes: 4

     Network            NextHop         MED        LocPrf     PrefVal Path/Ogn

* >i 10.1.1.0/24        3.3.3.3                    100        0       100?
* >  10.2.1.0/24        10.2.1.2        0                     32768   ?
*  e                    10.2.1.1        0                     0       65002?
* >  10.2.1.2/32        127.0.0.1       0                     32768   ?

#查看CE1 ping CE2
[CE1]ping 10.2.1.1
Ping 10.2.1.1 (10.2.1.1): 56 data bytes, press CTRL+C to break
56 bytes from 10.2.1.1: icmp_seq=0 ttl=253 time=1.000 ms
...
```





## 跨域VPN-OptionC（方式一）配置

#### 简介

> 由于PE之间不是直连，因此需要配置peer ebgp-max-hop命令，允许本地路由器同非直连网络上的邻居建立EBGP会话。

### 1.组网图

![](https://kim-1259075102.cos.ap-guangzhou.myqcloud.com/HCL-Picture/optionc1_v4.png)

### 2.组网需求

- Site 1和Site 2属于同一个VPN，Site 1通过AS 100的PE 1接入，Site 2通过AS 600的PE 2接入；
- 同一自治系统内的PE设备之间运行IS-IS作为IGP；
- PE 1与ASBR-PE 1间通过IBGP交换标签IPv4路由；
- PE 2与ASBR-PE 2间通过IBGP交换标签IPv4路由；
- PE 1与PE 2建立MP-EBGP对等体交换VPNv4路由；
- ASBR-PE 1和ASBR-PE 2上分别配置路由策略，对从对方接收的路由压入标签；
- ASBR-PE 1与ASBR-PE 2间通过EBGP交换标签IPv4路由。

### 3.配置步骤

```bash
(1)配置各接口的IP地址（略）
(2)配置CE1和CE2
#配置CE1与PE1建立EBGP对等体,并引入VPN路由
[CE1]bgp 65001
[CE1-bgp-default] peer 10.1.1.2 as-number 100
[CE1-bgp-default] address-family ipv4 unicast
[CE1-bgp-default-ipv4]  import-route direct
[CE1-bgp-default-ipv4]  peer 10.1.1.2 enable

#配置CE2与PE2建立EBGP对等体,并引入VPN路由
[CE2-bgp-default]bgp 65002
[CE2-bgp-default] peer 10.2.1.2 as-number 600
[CE2-bgp-default] address-family ipv4 unicast
[CE2-bgp-default-ipv4]  import-route direct
[CE2-bgp-default-ipv4]  peer 10.2.1.2 enable

(3)配置PE1
#在PE1运行OSPF
[PE1]ospf 1
[PE1-ospf-1] area 0.0.0.0
[PE1-ospf-1-area-0.0.0.0]  network 1.1.1.1 0.0.0.0
[PE1-ospf-1-area-0.0.0.0]  network 172.1.1.0 0.0.0.255

#配置LSR ID，使能MPLS和LDP
[PE1]mpls lsr-id 1.1.1.1
[PE1]mpls ldp
[PE1-ldp]quit
[PE1]interface GigabitEthernet0/0
[PE1-GigabitEthernet0/0] mpls enable
[PE1-GigabitEthernet0/0] mpls ldp enable

#创建VPN实例，名称为vpn1，为其配置RD和Route Target属性。
[PE1]ip vpn-instance vpn1
[PE1-vpn-instance-vpn1] route-distinguisher 11:11
[PE1-vpn-instance-vpn1] vpn-target 1:1 2:2 import-extcommunity
[PE1-vpn-instance-vpn1] vpn-target 1:1 export-extcommunity

#配置接口GigabitEthernet0/1与VPN实例vpn1绑定，并配置该接口的IP地址。
[PE1]interface GigabitEthernet0/1
[PE1-GigabitEthernet0/1] ip binding vpn-instance vpn1
[PE1-GigabitEthernet0/1] ip address 10.1.1.2 255.255.255.0

#配置PE1向IBGP对等体2.2.2.2发布标签路由及从2.2.2.2接收标签路由的能力
[PE1]bgp 100
[PE1-bgp-default] peer 2.2.2.2 as-number 100
[PE1-bgp-default] peer 2.2.2.2 connect-interface LoopBack0
[PE1-bgp-default] address-family ipv4 unicast
[PE1-bgp-default-ipv4]  peer 2.2.2.2 enable
[PE1-bgp-default-ipv4]  peer 2.2.2.2 label-route-capability

#配置PE1到EBGP对等体4.4.4.4的最大跳数为10,跳数需大于实际跳数

[PE1]bgp 100
[PE1-bgp-default] peer 4.4.4.4 as-number 600
[PE1-bgp-default] peer 4.4.4.4 connect-interface LoopBack0
[PE1-bgp-default] peer 4.4.4.4 ebgp-max-hop 10
[PE1-bgp-default-ipv4] address-family vpnv4
[PE1-bgp-default-vpnv4]  peer 4.4.4.4 enable

# 配置PE1与CE1建立EBGP对等体，将学习到的BGP路由添加到VPN实例的路由表中
[PE1]bgp 100
[PE1-bgp-default-vpnv4] ip vpn-instance vpn1
[PE1-bgp-default-vpn1]  peer 10.1.1.1 as-number 65001
[PE1-bgp-default-vpn1]  address-family ipv4 unicast
[PE1-bgp-default-ipv4-vpn1]   peer 10.1.1.1 enable

(4)配置ASBR-PE1
#在ASBR-PE1运行OSPF
[ASBR-PE1]ospf 1
[ASBR-PE1-ospf-1] area 0.0.0.0
[ASBR-PE1-ospf-1-area-0.0.0.0]  network 2.2.2.2 0.0.0.0
[ASBR-PE1-ospf-1-area-0.0.0.0]  network 172.1.1.0 0.0.0.255

#配置LSR ID，使能MPLS和LDP
[ASBR-PE1]mpls lsr-id 2.2.2.2
[ASBR-PE1]mpls ldp
[ASBR-PE1-ldp]quit
[ASBR-PE1]interface GigabitEthernet0/0
[ASBR-PE1-GigabitEthernet0/0] mpls enable
[ASBR-PE1-GigabitEthernet0/0] mpls ldp enable
[ASBR-PE1-GigabitEthernet0/0]interface GigabitEthernet0/1
[ASBR-PE1-GigabitEthernet0/1] mpls enable

#创建路由策略policy1和policy2
[ASBR-PE1]route-policy policy1 permit node 1
[ASBR-PE1-route-policy-policy1-1] apply mpls-label
[ASBR-PE1-route-policy-policy1-1]quit
[ASBR-PE1]route-policy policy2 permit node 1
[ASBR-PE1-route-policy-policy2-1] if-match mpls-label
[ASBR-PE1-route-policy-policy2-1] apply mpls-label

#在ASBR-PE1上运行BGP，并宣告1.1.1.1/32的路由
[ASBR-PE1]bgp 100
[ASBR-PE1-bgp-default] address-family ipv4 unicast
[ASBR-PE1-bgp-default-ipv4]  network 1.1.1.1 255.255.255.255


#对向IBGP对等体PE1发布的路由应用已配置的路由策略policy2
[ASBR-PE1]bgp 100
[ASBR-PE1-bgp-default] peer 1.1.1.1 as-number 100
[ASBR-PE1-bgp-default] peer 1.1.1.1 connect-interface LoopBack0
[ASBR-PE1-bgp-default] address-family ipv4 unicast
[ASBR-PE1-bgp-default-ipv4]  peer 1.1.1.1 enable
[ASBR-PE1-bgp-default-ipv4]  peer 1.1.1.1 route-policy policy2 export
[ASBR-PE1-bgp-default-ipv4]  peer 1.1.1.1 next-hop-local
#向IBGP对等体PE1发布标签路由及从1.1.1.1接收标签路由的能力。
[ASBR-PE1-bgp-default-ipv4]  peer 1.1.1.1 label-route-capability

#对向EBGP对等体ASBR-PE2发布的路由应用已配置的路由策略policy1
[ASBR-PE1]bgp 100
[ASBR-PE1-bgp-default] peer 192.1.1.2 as-number 600
[ASBR-PE1-bgp-default] address-family ipv4 unicast
[ASBR-PE1-bgp-default-ipv4]  peer 192.1.1.2 enable
[ASBR-PE1-bgp-default-ipv4]  peer 192.1.1.2 route-policy policy1 export
#向EBGP对等体发布标签路由及从192.1.1.2接受标签路由的能力
[ASBR-PE1-bgp-default-ipv4]  peer 192.1.1.2 label-route-capability

(5)配置ASBR-PE2
#在ASBR-PE2运行OSPF
[ASBR-PE2]ospf 1
[ASBR-PE2-ospf-1] area 0.0.0.0
[ASBR-PE2-ospf-1-area-0.0.0.0]  network 3.3.3.3 0.0.0.0
[ASBR-PE2-ospf-1-area-0.0.0.0]  network 162.1.1.0 0.0.0.255

#配置LSR ID，使能MPLS和LDP
[ASBR-PE2]mpls lsr-id 3.3.3.3
[ASBR-PE2]mpls ldp
[ASBR-PE2-ldp]quit
[ASBR-PE2]interface GigabitEthernet0/0
[ASBR-PE2-GigabitEthernet0/0] mpls enable
[ASBR-PE2-GigabitEthernet0/0] mpls ldp enable
[ASBR-PE2-GigabitEthernet0/0]interface GigabitEthernet0/1
[ASBR-PE2-GigabitEthernet0/1]mpls enable

#创建路由策略policy1和policy2
[ASBR-PE2]route-policy policy1 permit node 1
[ASBR-PE2-route-policy-policy1-1] apply mpls-label
[ASBR-PE2-route-policy-policy1-1]quit
[ASBR-PE2]route-policy policy2 permit node 1
[ASBR-PE2-route-policy-policy2-1] if-match mpls-label
[ASBR-PE2-route-policy-policy2-1] apply mpls-label

#在ASBR-PE2上运行BGP，并宣告4.4.4.4/32的路由
[ASBR-PE2]bgp 600
[ASBR-PE2-bgp-default] address-family ipv4 unicast
[ASBR-PE2-bgp-default-ipv4]  network 4.4.4.4 255.255.255.255

#对向IBGP对等体PE2发布的路由应用已配置的路由策略policy2
[ASBR-PE2]bgp 600
[ASBR-PE2-bgp-default] peer 4.4.4.4 as-number 600
[ASBR-PE2-bgp-default] peer 4.4.4.4 connect-interface LoopBack0
[ASBR-PE2-bgp-default] address-family ipv4 unicast
[ASBR-PE2-bgp-default-ipv4]  peer 4.4.4.4 enable
[ASBR-PE2-bgp-default-ipv4]  peer 4.4.4.4 route-policy policy2 export
[ASBR-PE2-bgp-default-ipv4]  peer 4.4.4.4 next-hop-local
#向IBGP对等体PE2发布标签路由及从1.1.1.1接收标签路由的能力。
[ASBR-PE2-bgp-default-ipv4]  peer 4.4.4.4 label-route-capability

#对向EBGP对等体ASBR-PE1发布的路由应用已配置的路由策略policy1
[ASBR-PE2]bgp 600
[ASBR-PE2-bgp-default] peer 192.1.1.1 as-number 100
[ASBR-PE2-bgp-default] address-family ipv4 unicast
[ASBR-PE2-bgp-default-ipv4]  peer 192.1.1.1 enable
[ASBR-PE2-bgp-default-ipv4]  peer 192.1.1.1 route-policy policy1 export
#向EBGP对等体发布标签路由及从192.1.1.1接受标签路由的能力
[ASBR-PE2-bgp-default-ipv4]  peer 192.1.1.1 label-route-capability

(6)配置PE2
#在PE2运行OSPF
[PE2]ospf 1
[PE2-ospf-1] area 0.0.0.0
[PE2-ospf-1-area-0.0.0.0]  network 4.4.4.4 0.0.0.0
[PE2-ospf-1-area-0.0.0.0]  network 162.1.1.0 0.0.0.255

#配置LSR ID，使能MPLS和LDP
[PE2] mpls lsr-id 4.4.4.4
[PE2] mpls  ldp
[PE2-ldp]quit
[PE2]interface GigabitEthernet0/0
[PE2-GigabitEthernet0/0] mpls enable
[PE2-GigabitEthernet0/0] mpls ldp enable

#创建VPN实例，名称为vpn1，为其配置RD和Route Target属性。
[PE2]ip vpn-instance vpn1
[PE2-vpn-instance-vpn1] route-distinguisher 12:12
[PE2-vpn-instance-vpn1] vpn-target 1:1 2:2 import-extcommunity
[PE2-vpn-instance-vpn1] vpn-target 2:2 export-extcommunity

#配置接口GigabitEthernet0/1与VPN实例vpn1绑定，并配置该接口的IP地址。
[PE2]interface GigabitEthernet0/1
[PE2-GigabitEthernet0/1] ip binding vpn-instance vpn1
[PE2-GigabitEthernet0/1] ip address 10.2.1.2 255.255.255.0

#配置PE2向IBGP对等体ASBR-PE2发布标签路由及从3.3.3.3接收标签路由的能力
[PE2]bgp 600
[PE2-bgp-default] peer 3.3.3.3 as-number 600
[PE2-bgp-default] peer 3.3.3.3 connect-interface LoopBack0
[PE2-bgp-default] address-family ipv4 unicast
[PE2-bgp-default-ipv4]  peer 3.3.3.3 enable
[PE2-bgp-default-ipv4]  peer 3.3.3.3 label-route-capability

#配置PE2到EBGP对等体1.1.1.1的最大跳数为10,跳数需大于实际跳数
[PE2]bgp 600
[PE2-bgp-default] peer 1.1.1.1 as-number 100
[PE2-bgp-default] peer 1.1.1.1 connect-interface LoopBack0
[PE2-bgp-default] peer 1.1.1.1 ebgp-max-hop 10
[PE2-bgp-default-ipv4] address-family vpnv4
[PE2-bgp-default-vpnv4]  peer 1.1.1.1 enable

# 配置PE2与CE2建立EBGP对等体，将学习到的BGP路由添加到VPN实例的路由表中
[PE2]bgp 600
[PE2-bgp-default-vpnv4] ip vpn-instance vpn1
[PE2-bgp-default-vpn1]  peer 10.2.1.1 as-number 65002
[PE2-bgp-default-vpn1]  address-family ipv4 unicast
[PE2-bgp-default-ipv4-vpn1]   peer 10.2.1.1 enable
```

### 4.验证配置

```bash
#查看PE1和PE2的BGP路由表
[PE1]display bgp routing-table vpnv4
...

     Network            NextHop         MED        LocPrf     PrefVal Path/Ogn

* >e 10.1.1.0/24        10.1.1.1        0                     0       65001?
* >e 10.2.1.0/24        4.4.4.4                               0       600 65002?

 Route distinguisher: 12:12
 Total number of routes: 1

     Network            NextHop         MED        LocPrf     PrefVal Path/Ogn

* >e 10.2.1.0/24        4.4.4.4                               0       600 65002?


[PE2]display  bgp routing-table vpnv4
...

     Network            NextHop         MED        LocPrf     PrefVal Path/Ogn

* >e 10.1.1.0/24        1.1.1.1                               0       100 65001?

 Route distinguisher: 12:12(vpn1)
 Total number of routes: 2

     Network            NextHop         MED        LocPrf     PrefVal Path/Ogn

* >e 10.1.1.0/24        1.1.1.1                               0       100 65001?
* >e 10.2.1.0/24        10.2.1.1        0                     0       65002?

#查看CE1 ping CE2
[CE1]ping 10.2.1.1
Ping 10.2.1.1 (10.2.1.1): 56 data bytes, press CTRL+C to break
56 bytes from 10.2.1.1: icmp_seq=0 ttl=253 time=1.000 ms
...
```



## 跨域VPN-OptionC（方式二）配置

### 1.组网图

![](https://kim-1259075102.cos.ap-guangzhou.myqcloud.com/HCL-Picture/optionc2_v4.png)

| 设备     | 接口      | IP地址       | 设备     | 接口      | IP地址       |
| -------- | --------- | ------------ | -------- | --------- | ------------ |
| CE1      | GE0/1     | 10.1.1.1/24  | CE2      | GE0/1     | 10.2.1.1/24  |
| PE1      | GE0/0     | 172.1.1.2/24 | PE2      | GE0/0     | 162.1.1.2/24 |
|          | GE0/1     | 10.1.1.2/24  |          | GE0/1     | 10.2.1.2/24  |
|          | loopback0 | 1.1.1.1/32   |          | loopback0 | 4.4.4.4/32   |
| ASBR-PE1 | GE0/0     | 172.1.1.1/24 | ASBR-PE2 | GE0/0     | 162.1.1.1/24 |
|          | GE0/1     | 192.1.1.1/24 |          | GE0/1     | 192.1.1.2/24 |
|          | loopback0 | 2.2.2.2/32   |          | loopback0 | 3.3.3.3/32   |

### 2.组网需求

- Site 1和Site 2属于同一个VPN，Site 1通过AS 100的PE 1接入，Site 2通过AS 600的PE 2接入；
- 同一自治系统内的PE设备之间运行IS-IS作为IGP；
- PE 1与PE 2建立MP-EBGP对等体交换VPNv4路由；
- ASBR-PE 1和ASBR-PE 2上分别配置路由策略，对从对方接收的路由压入标签；
- ASBR-PE 1与ASBR-PE 2间通过EBGP交换标签IPv4路由。
- ASBR-PE 1和ASBR-PE 2上将IGP路由和BGP路由互相引入。

### 3.配置步骤

```bash
(1)配置各接口的IP地址（略）
(2)配置CE1和CE2
#配置CE1与PE1建立EBGP对等体,并引入VPN路由
[CE1]bgp 65001
[CE1-bgp-default] peer 10.1.1.2 as-number 100
[CE1-bgp-default] address-family ipv4 unicast
[CE1-bgp-default-ipv4]  import-route direct
[CE1-bgp-default-ipv4]  peer 10.1.1.2 enable

#配置CE2与PE2建立EBGP对等体,并引入VPN路由
[CE2-bgp-default]bgp 65002
[CE2-bgp-default] peer 10.2.1.2 as-number 600
[CE2-bgp-default] address-family ipv4 unicast
[CE2-bgp-default-ipv4]  import-route direct
[CE2-bgp-default-ipv4]  peer 10.2.1.2 enable

(3)配置PE1
#在PE1运行OSPF
[PE1]ospf 1
[PE1-ospf-1] area 0.0.0.0
[PE1-ospf-1-area-0.0.0.0]  network 1.1.1.1 0.0.0.0
[PE1-ospf-1-area-0.0.0.0]  network 172.1.1.0 0.0.0.255

#配置LSR ID，使能MPLS和LDP
[PE1]mpls lsr-id 1.1.1.1
[PE1]mpls ldp
[PE1-ldp]quit
[PE1]interface GigabitEthernet0/0
[PE1-GigabitEthernet0/0] mpls enable
[PE1-GigabitEthernet0/0] mpls ldp enable

#创建VPN实例，名称为vpn1，为其配置RD和Route Target属性。
[PE1]ip vpn-instance vpn1
[PE1-vpn-instance-vpn1] route-distinguisher 11:11
[PE1-vpn-instance-vpn1] vpn-target 1:1 2:2 import-extcommunity
[PE1-vpn-instance-vpn1] vpn-target 1:1 export-extcommunity

#配置接口GigabitEthernet0/1与VPN实例vpn1绑定，并配置该接口的IP地址。
[PE1]interface GigabitEthernet0/1
[PE1-GigabitEthernet0/1] ip binding vpn-instance vpn1
[PE1-GigabitEthernet0/1] ip address 10.1.1.2 255.255.255.0

#配置PE1到EBGP对等体4.4.4.4的最大跳数为10,跳数需大于实际跳数
[PE1]bgp 100
[PE1-bgp-default] peer 4.4.4.4 as-number 600
[PE1-bgp-default] peer 4.4.4.4 connect-interface LoopBack0
[PE1-bgp-default] peer 4.4.4.4 ebgp-max-hop 10
[PE1-bgp-default] address-family vpnv4
[PE1-bgp-default-vpnv4]  peer 4.4.4.4 enable

# 配置PE1与CE1建立EBGP对等体，将学习到的BGP路由添加到VPN实例的路由表中
[PE1-bgp-default-vpnv4] ip vpn-instance vpn1
[PE1-bgp-default-vpn1]  peer 10.1.1.1 as-number 65001
[PE1-bgp-default-vpn1]  address-family ipv4 unicast
[PE1-bgp-default-ipv4-vpn1]   peer 10.1.1.1 enable

(4)配置ASBR-PE1
#在ASBR-PE1运行OSPF，并引入BGP路由
[ASBR-PE1]ospf 1
[ASBR-PE1-ospf-1] import-route bgp
[ASBR-PE1-ospf-1] area 0.0.0.0
[ASBR-PE1-ospf-1-area-0.0.0.0]  network 2.2.2.2 0.0.0.0
[ASBR-PE1-ospf-1-area-0.0.0.0]  network 172.1.1.0 0.0.0.255

#配置LSR ID，使能MPLS和LDP
[ASBR-PE1]mpls lsr-id 2.2.2.2
[ASBR-PE1]mpls ldp
[ASBR-PE1-ldp]quit
[ASBR-PE1]interface GigabitEthernet0/0
[ASBR-PE1-GigabitEthernet0/0] mpls enable
[ASBR-PE1-GigabitEthernet0/0] mpls ldp enable
[ASBR-PE1-GigabitEthernet0/0]interface GigabitEthernet0/1
[ASBR-PE1-GigabitEthernet0/1] mpls enable

#创建路由策略
[ASBR-PE1]route-policy policy1 permit node 1
[ASBR-PE1-route-policy-policy1-1] apply mpls-label

#在ASBR-PE1上运行BGP，引入OSPF的路由
[ASBR-PE1]bgp 100
[ASBR-PE1-bgp-default] address-family ipv4 unicast
[ASBR-PE1-bgp-default-ipv4]  import-route ospf 1

#对向EBGP对等体ASBR-PE2发布的路由应用已配置的路由策略policy1
[ASBR-PE1]bgp 100
[ASBR-PE1-bgp-default] peer 192.1.1.2 as-number 600
[ASBR-PE1-bgp-default] address-family ipv4 unicast
[ASBR-PE1-bgp-default-ipv4]  peer 192.1.1.2 enable
[ASBR-PE1-bgp-default-ipv4]  peer 192.1.1.2 route-policy policy1 export
#向EBGP对等体发布标签路由及从192.1.1.2接受标签路由的能力
[ASBR-PE1-bgp-default-ipv4]  peer 192.1.1.2 label-route-capability

(5)配置ASBR-PE2
#在ASBR-PE2运行OSPF，并引入BGP路由
[ASBR-PE2] ospf 1
[ASBR-PE2-ospf-1] import-route bgp
[ASBR-PE2-ospf-1] area 0.0.0.0
[ASBR-PE2-ospf-1-area-0.0.0.0]  network 3.3.3.3 0.0.0.0
[ASBR-PE2-ospf-1-area-0.0.0.0]  network 162.1.1.0 0.0.0.255

#配置LSR ID，使能MPLS和LDP
[ASBR-PE2]mpls lsr-id 3.3.3.3
[ASBR-PE2]mpls ldp
[ASBR-PE2-ldp]quit
[ASBR-PE2]interface GigabitEthernet0/0
[ASBR-PE2-GigabitEthernet0/0] mpls enable
[ASBR-PE2-GigabitEthernet0/0] mpls ldp enable
[ASBR-PE2-GigabitEthernet0/0]interface GigabitEthernet0/1
[ASBR-PE2-GigabitEthernet0/1]mpls enable

#创建路由策略
[ASBR-PE2]route-policy policy1 permit node 1
[ASBR-PE2-route-policy-policy1-1] apply mpls-label

#在ASBR-PE2上运行BGP，引入OSPF的路由
[ASBR-PE2]bgp 600
[ASBR-PE2-bgp-default] address-family ipv4 unicast
[ASBR-PE2-bgp-default-ipv4]  import-route ospf 1

#对向EBGP对等体ASBR-PE1发布的路由应用已配置的路由策略policy1
[ASBR-PE2]bgp 600
[ASBR-PE2-bgp-default] peer 192.1.1.1 as-number 100
[ASBR-PE2-bgp-default] address-family ipv4 unicast
[ASBR-PE2-bgp-default-ipv4]  peer 192.1.1.1 enable
[ASBR-PE2-bgp-default-ipv4]  peer 192.1.1.1 route-policy policy1 export

#向EBGP对等体发布标签路由及从192.1.1.1接受标签路由的能力
[ASBR-PE2-bgp-default-ipv4]  peer 192.1.1.1 label-route-capability

(6)配置PE2
#在PE2运行OSPF
[PE2]ospf 1
[PE2-ospf-1] area 0.0.0.0
[PE2-ospf-1-area-0.0.0.0]  network 4.4.4.4 0.0.0.0
[PE2-ospf-1-area-0.0.0.0]  network 162.1.1.0 0.0.0.255

#配置LSR ID，使能MPLS和LDP
[PE2]mpls lsr-id 4.4.4.4
[PE2]mpls ldp
[PE2-ldp]quit
[PE2]interface GigabitEthernet0/0
[PE2-GigabitEthernet0/0] mpls enable
[PE2-GigabitEthernet0/0] mpls ldp enable

#创建VPN实例，名称为vpn1，为其配置RD和Route Target属性。
[PE2]ip vpn-instance vpn1
[PE2-vpn-instance-vpn1] route-distinguisher 12:12
[PE2-vpn-instance-vpn1] vpn-target 1:1 2:2 import-extcommunity
[PE2-vpn-instance-vpn1] vpn-target 2:2 export-extcommunity

#配置接口GigabitEthernet0/1与VPN实例vpn1绑定，并配置该接口的IP地址。
[PE2]interface GigabitEthernet0/1
[PE2-GigabitEthernet0/1] ip binding vpn-instance vpn1
[PE2-GigabitEthernet0/1] ip address 10.2.1.2 255.255.255.0

#配置PE1到EBGP对等体1.1.1.1的最大跳数为10,跳数需大于实际跳数
[PE2]bgp 600
[PE2-bgp-default] peer 1.1.1.1 as-number 100
[PE2-bgp-default] peer 1.1.1.1 connect-interface LoopBack0
[PE2-bgp-default] peer 1.1.1.1 ebgp-max-hop 10
[PE2-bgp-default] address-family vpnv4
[PE2-bgp-default-vpnv4]  peer 1.1.1.1 enable

# 配置PE1与CE1建立EBGP对等体，将学习到的BGP路由添加到VPN实例的路由表中
[PE2-bgp-default-vpnv4] ip vpn-instance vpn1
[PE2-bgp-default-vpn1]  peer 10.2.1.1 as-number 65002
[PE2-bgp-default-vpn1]  address-family ipv4 unicast
[PE2-bgp-default-ipv4-vpn1]   peer 10.2.1.1 enable
```

### 4.验证配置

```bash
#查看PE1与CE1的BGP邻居关系
[PE1]display  bgp peer  ipv4 vpn-instance vpn1
...
  Peer                    AS  MsgRcvd  MsgSent OutQ PrefRcv Up/Down  State

  10.1.1.1             65001       89       81    0       1 01:06:10 Established

#查看PE2与CE2的BGP邻居关系
[PE2]display bgp peer ipv4 vpn-instance vpn1
...
  Peer                    AS  MsgRcvd  MsgSent OutQ PrefRcv Up/Down  State

  10.2.1.1             65002       91       82    0       1 01:07:38 Established

#查看PE1与PE2的BGP邻居关系
[PE1]display bgp peer  vpnv4
...
  Peer                    AS  MsgRcvd  MsgSent OutQ PrefRcv Up/Down  State

  4.4.4.4                600       32       37    0       1 00:25:30 Established

[PE2]display bgp peer  vpnv4
...
  Peer                    AS  MsgRcvd  MsgSent OutQ PrefRcv Up/Down  State

  1.1.1.1                100       37       32    0       1 00:25:47 Established

#查看PE1和PE2的BGP路由表
[PE1]display  bgp routing-table  ipv4 vpn-instance vpn1
...

     Network            NextHop         MED        LocPrf     PrefVal Path/Ogn

* >e 10.1.1.0/24        10.1.1.1        0                     0       65001?
* >e 10.2.1.0/24        4.4.4.4                               0       600 65002?

[PE2]display  bgp routing-table  ipv4 vpn-instance vpn1
...
     Network            NextHop         MED        LocPrf     PrefVal Path/Ogn

* >e 10.1.1.0/24        1.1.1.1                               0       100 65001?
* >e 10.2.1.0/24        10.2.1.1        0                     0       65002?

#CE1 ping CE2
[CE1]ping 10.2.1.1
Ping 10.2.1.1 (10.2.1.1): 56 data bytes, press CTRL+C to break
56 bytes from 10.2.1.1: icmp_seq=0 ttl=253 time=1.000 ms
...
```





