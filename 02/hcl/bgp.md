## BGP团体配置

#### 简介

> 缺省情况下，本地路由器不向对等体/对等体组发布团体属性和扩展团体属性。如果接收到的路由中携带团体属性或扩展团体属性，则本地路由器删除该团体属性或扩展团体属性后，再将路由发布给对等体/对等体组。
>
> 通过本配置可以允许本地路由器在向对等体发布路由时携带团体属性或扩展团体属性，以便根据团体属性或扩展团体属性对路由进行过滤和控制。本配置和路由策略配合使用，可以灵活地控制路由中携带的团体属性和扩展团体属性值，例如在路由中添加团体属性或扩展团体属性、修改路由中原有的团体属性或扩展团体属性值。

### 1.组网图

<img src="https://kim-1259075102.cos.ap-guangzhou.myqcloud.com/HCL-Picture/bgp_community.png" style="zoom: 80%;" />

| 设备    | 接口      | IP地址       | 设备    | 接口      | IP地址       |
| ------- | --------- | ------------ | ------- | --------- | ------------ |
| RouterA | GE0/0     | 200.1.2.1/24 | RouterC | GE0/0     | 200.1.3.2/24 |
|         | Loopback0 | 1.1.1.1/32   |         | Loopback0 | 3.3.3.3/32   |
|         | Loopback1 | 9.1.1.1/24   |         |           |              |
| RouterB | GE0/0     | 200.1.2.2/24 |         |           |              |
|         | GE0/1     | 200.1.3.1/24 |         |           |              |
|         | Loopback0 | 2.2.2.2/32   |         |           |              |
|         | Loopback1 | 9.1.1.1/24   |         |           |              |



### 2.组网需求

- Router B分别与Router A、Router C之间建立EBGP连接。

- 通过在Router A上配置NO_EXPORT团体属性，使得AS 10发布到AS 20中的路由，不会再被AS 20发布到其他AS。

### 3.配置步骤

```bash
(1)配置各接口的IP地址（略）
(2)配置EBGP
#配置RouterA
[RouterA] bgp 10
[RouterA-bgp-default] router-id 1.1.1.1
[RouterA-bgp-default] peer 200.1.2.2 as-number 20
[RouterA-bgp-default] address-family ipv4 unicast
[RouterA-bgp-default-ipv4] peer 200.1.2.2 enable
[RouterA-bgp-default-ipv4] network 9.1.1.0 255.255.255.0
[RouterA-bgp-default-ipv4] quit
[RouterA-bgp-default] quit

#配置RouterB
[RouterB] bgp 20
[RouterB-bgp-default] router-id 2.2.2.2
[RouterB-bgp-default] peer 200.1.2.1 as-number 10
[RouterB-bgp-default] peer 200.1.3.2 as-number 30
[RouterB-bgp-default] address-family ipv4 unicast
[RouterB-bgp-default-ipv4] peer 200.1.2.1 enable
[RouterB-bgp-default-ipv4] peer 200.1.3.2 enable
[RouterB-bgp-default-ipv4] quit
[RouterB-bgp-default] quit

#配置RouterC
[RouterC] bgp 30
[RouterC-bgp-default] router-id 3.3.3.3
[RouterC-bgp-default] peer 200.1.3.1 as-number 20
[RouterC-bgp-default] address-family ipv4 unicast
[RouterC-bgp-default-ipv4] peer 200.1.3.1 enable
[RouterC-bgp-default-ipv4] quit
[RouterC-bgp-default] quit


#查看RouterB的路由表
[RouterB]display bgp  routing-table ipv4 9.1.1.0

 BGP local router ID: 2.2.2.2
 Local AS number: 20

 Paths:   1 available, 1 best

 BGP routing table information of 9.1.1.0/24:
 From            : 200.1.2.1 (1.1.1.1)
 Rely nexthop    : 200.1.2.1
 Original nexthop: 200.1.2.1
 OutLabel        : NULL
 AS-path         : 10
 Origin          : igp
 Attribute value : pref-val 0
 State           : valid, external, best
...

#查看RouterB的路由发送信息
[RouterB] display bgp routing-table ipv4 9.1.1.0 advertise-info
...

 BGP routing table information of 9.1.1.0/24(TxPathID:0):
 Advertised to peers (1 in total):
    200.1.3.2
可以看出，Router B能够把到达目的地址9.1.1.0/24的路由通过BGP发布出去。

# 查看RouterC的BGP路由表
[RouterC] display bgp routing-table ipv4
...
     Network            NextHop         MED        LocPrf     PrefVal Path/Ogn
* >e 9.1.1.0/24         200.1.3.1                             0       20 10i
可以看出，RouterC从RouterB那里学到了目的地址为9.1.1.0/24的路由。

(3)配置BGP团体属性
# 配置路由策略。
[RouterA] route-policy comm_policy permit node 0
[RouterA-route-policy-comm_policy-0] apply community no-export
[RouterA-route-policy-comm_policy-0] quit

# 应用路由策略。
[RouterA] bgp 10
[RouterA-bgp-default] address-family ipv4 unicast
[RouterA-bgp-default-ipv4] peer 200.1.2.2 route-policy comm_policy export
[RouterA-bgp-default-ipv4] peer 200.1.2.2 advertise-community

```

### 4.验证配置

```bash
#查看RouterB的路由表。
[RouterB]display bgp routing-table ipv4 9.1.1.0

 BGP local router ID: 2.2.2.2
 Local AS number: 20

 Paths:   1 available, 1 best

 BGP routing table information of 9.1.1.0/24:
 From            : 200.1.2.1 (1.1.1.1)
 Rely nexthop    : 200.1.2.1
 Original nexthop: 200.1.2.1
 OutLabel        : NULL
 Community       : No-Export
 AS-path         : 10
 Origin          : igp
 Attribute value : MED 0, pref-val 0
 State           : valid, external, best
...

#查看RouterB的路由发送信息。
[RouterB]display bgp routing-table ipv4 9.1.1.0 advertise-info
...

 BGP routing table information of 9.1.1.0/24:
 Not advertised to any peers yet

#查看RouterC的BGP路由表。
[RouterC]display bgp routing-table ipv4

 Total number of routes: 0
```



## BGP路由反射器配置

#### 简介

> 如果同一个AS内有多个BGP路由器，为了减少在同一AS内建立的IBGP连接数，可以将一台BGP路由器配置为路由反射器，其他路由器作为前者的客户机，使反射器和它的客户机组成为一个集群。集群内部通过路由反射器在客户机之间反射路由，各客户机之间不需要建立BGP连接即可交换路由信息。
> 为了增加网络的可靠性和防止单点故障，可以在一个集群中配置一个以上的路由反射器，这时，网络管理员必须给位于相同集群中的每个路由反射器配置相同的集群ID，以避免路由环路。
>
> 当一台路由反射器可能连接网络中的多个集群时，可以为不同对等体/对等体组指定集群ID，以便对路由反射进行更精细控制。
> BGP路由反射功能仅需在作为反射器的设备上进行配置，其他设备不需要感知本机在反射功能中作为客户机或是非客户机。
>
> 设备被配置为路由反射器后，发布路由的规则如下：
> - 将从IBGP对等体中非客户机设备收到的路由，发布给本反射器的所有客户机；
> - 将从IBGP对等体中客户机收到的路由，发布给本反射器所有的非客户机和客户机；
> - 将从所有EBGP对等体收到的路由，发布给本反射器所有的非客户机和客户机。
>
> 规则总结：非非不传，其它都传，指非客户端不传非客户端

### 1.组网图

![](https://kim-1259075102.cos.ap-guangzhou.myqcloud.com/HCL-Picture/bgp_reflector.png)

| 设备    | 接口      | IP地址       | 设备    | 接口      | IP地址       |
| ------- | --------- | ------------ | ------- | --------- | ------------ |
| RouterA | GE0/0     | 192.1.1.1/24 | RouterC | GE0/0     | 194.1.1.1/24 |
|         | Loopback0 | 1.1.1.1/32   |         | GE0/1     | 193.1.1.1/24 |
|         | Loopback1 | 20.1.1.1/8   |         | Loopback0 | 3.3.3.3/32S  |
| RouterB | GE0/0     | 192.1.1.2/24 | RouterD | GE0/0     | 194.1.1.2/24 |
|         | GE0/1     | 193.1.1.2/24 |         | Loopback0 | 4.4.4.4/32   |
|         | Loopback0 | 2.2.2.2/32   |         |           |              |

### 2.组网需求

- 所有路由器运行BGP协议，Router A与Router B建立EBGP连接，Router C与Router B和Router D之间建立IBGP连接。
- Router C作为路由反射器，Router B和Router D为Router C的客户机。
- Router D能够通过Router C学到路由20.0.0.0/8。

### 3.配置步骤

```bash
(1)配置各接口的IP地址（略）
(2)配置AS200内的OSPF
#配置RouterB
[RouterB]ospf 1 router-id 2.2.2.2
[RouterB-ospf-1] area 0.0.0.0
[RouterB-ospf-1-area-0.0.0.0]  network 2.2.2.2 0.0.0.0
[RouterB-ospf-1-area-0.0.0.0]  network 193.1.1.0 0.0.0.255

#配置RouterC
RouterC]ospf 1 router-id 3.3.3.3
[RouterC-ospf-1] area 0.0.0.0
[RouterC-ospf-1-area-0.0.0.0]  network 3.3.3.3 0.0.0.0
[RouterC-ospf-1-area-0.0.0.0]  network 193.1.1.0 0.0.0.255
[RouterC-ospf-1-area-0.0.0.0]  network 194.1.1.0 0.0.0.255

#配置RouterD
[RouterD]ospf 1
[RouterD-ospf-1] area 0.0.0.0
[RouterD-ospf-1-area-0.0.0.0]  network 4.4.4.4 0.0.0.0
[RouterD-ospf-1-area-0.0.0.0]  network 194.1.1.0 0.0.0.255

(3)配置BGP
#配置RouterA
[RouterA] bgp 100
[RouterA-bgp-default] router-id 1.1.1.1
[RouterA-bgp-default] peer 192.1.1.2 as-number 200
[RouterA-bgp-default] address-family ipv4 unicast
[RouterA-bgp-default-ipv4] peer 192.1.1.2 enable

#通告20.0.0.0/8网段路由到BGP路由表中
[RouterA-bgp-default-ipv4] network 20.0.0.0
[RouterA-bgp-default-ipv4] quit
[RouterA-bgp-default] quit

#配置RouterB
[RouterB]bgp 200
[RouterB-bgp-default] router-id 2.2.2.2
[RouterB-bgp-default] peer 3.3.3.3 as-number 200
[RouterB-bgp-default] peer 3.3.3.3 connect-interface LoopBack0
[RouterB-bgp-default] peer 192.1.1.1 as-number 100
[RouterB-bgp-default] address-family ipv4 unicast
[RouterB-bgp-default-ipv4]  peer 3.3.3.3 enable
[RouterB-bgp-default-ipv4]  peer 3.3.3.3 next-hop-local
[RouterB-bgp-default-ipv4]  peer 192.1.1.1 enable

# 配置Router C。
[RouterC]bgp 200
[RouterC-bgp-default] router-id 3.3.3.3
[RouterC-bgp-default] peer 2.2.2.2 as-number 200
[RouterC-bgp-default] peer 2.2.2.2 connect-interface LoopBack0
[RouterC-bgp-default] peer 4.4.4.4 as-number 200
[RouterC-bgp-default] peer 4.4.4.4 connect-interface LoopBack0
[RouterC-bgp-default] address-family ipv4 unicast
[RouterC-bgp-default-ipv4]  peer 2.2.2.2 enable
[RouterC-bgp-default-ipv4]  peer 4.4.4.4 enable

#配置Router D。
[RouterD]bgp 200
[RouterD-bgp-default] router-id 4.4.4.4
[RouterD-bgp-default] peer 3.3.3.3 as-number 200
[RouterD-bgp-default] peer 3.3.3.3 connect-interface LoopBack0
[RouterD-bgp-default] address-family ipv4 unicast
[RouterD-bgp-default-ipv4]  peer 3.3.3.3 enable

(4)配置路由反射器
#配置RouterC
[RouterC]bgp 200
[RouterC-bgp-default] address-family ipv4 unicast
[RouterC-bgp-default-ipv4]  peer 2.2.2.2 reflect-client
[RouterC-bgp-default-ipv4]  peer 4.4.4.4 reflect-client
```

### 4.验证配置

```bash
#查看RouterB的BGP路由表
[RouterB]display bgp routing-table ipv4
...

     Network            NextHop         MED        LocPrf     PrefVal Path/Ogn

* >e 20.1.1.0/24        192.1.1.1       0                     0       100i

#查看RouterD的BGP路由表
[RouterD]display  bgp routing-table ipv4
...

     Network            NextHop         MED        LocPrf     PrefVal Path/Ogn

* >i 20.1.1.0/24        2.2.2.2         0          100        0       100i
由下一跳可以看出，RouterD从Router C已经学到了20.0.0.0/8路由。
```





## BGP联盟配置

#### 简介

> 联盟是处理AS内部的IBGP网络连接激增的另一种方法，它将一个自治系统划分为若干个子自治系统，每个子自治系统内部的IBGP对等体建立全连接关系，子自治系统之间建立EBGP连接关系。

### 1.组网图

![](https://kim-1259075102.cos.ap-guangzhou.myqcloud.com/HCL-Picture/bgp_confederation.png)

| 设备     | 接口      | IP地址       | 设备     | 接口      | IP地址       |
| -------- | --------- | ------------ | -------- | --------- | ------------ |
| Router A | GE0/0     | 200.1.1.1/24 | Router D | GE0/0     | 10.1.5.1/24  |
|          | GE0/1     | 10.1.3.1/24  |          | GE0/1     | 10.1.3.2/24  |
|          | GE0/2     | 10.1.4.1/24  |          | Loopback0 | 4.4.4.4/32   |
|          | GE5/0     | 10.1.1.1/24  | Router E | GE0/0     | 10.1.5.2/24  |
|          | GE5/1     | 10.1.2.1/24  |          | GE0/1     | 10.1.4.2/24  |
|          | Loopback0 | 1.1.1.1/32   |          | Loopback0 | 5.5.5.5/32   |
| Router B | GE0/0     | 10.1.1.2/24  | Router F | GE0/0     | 200.1.1.2/24 |
|          | Loopback0 | 2.2.2.2/32   |          | Loopback0 | 9.1.1.1/24   |
| Router C | GE0/0     | 10.1.2.2/24  |          |           |              |
|          | Loopback0 | 3.3.3.3/32   |          |           |              |

### 2.组网需求

- AS 200中有多台BGP路由器，为了减少IBGP的连接数，现将他们划分为3个子自治系统：AS 65001、AS 65002和AS 65003。其中AS 65001内的三台路由器建立IBGP全连接。


### 3.配置步骤

```bash
(1)配置各接口的IP地址（略）
(2)配置配置BGP联盟
#配置RouterA
[RouterA] bgp 65001
[RouterA-bgp-default] router-id 1.1.1.1
[RouterA-bgp-default] confederation id 200
[RouterA-bgp-default] confederation peer-as 65002 65003
[RouterA-bgp-default] peer 10.1.1.2 as-number 65002
[RouterA-bgp-default] peer 10.1.2.2 as-number 65003
[RouterA-bgp-default] address-family ipv4 unicast
[RouterA-bgp-default-ipv4] peer 10.1.1.2 enable
[RouterA-bgp-default-ipv4] peer 10.1.2.2 enable
[RouterA-bgp-default-ipv4] peer 10.1.1.2 next-hop-local
[RouterA-bgp-default-ipv4] peer 10.1.2.2 next-hop-local
[RouterA-bgp-default-ipv4] quit
[RouterA-bgp-default] quit

#配置RouterB
[RouterB] bgp 65002
[RouterB-bgp-default] router-id 2.2.2.2
[RouterB-bgp-default] confederation id 200
[RouterB-bgp-default] confederation peer-as 65001 65003
[RouterB-bgp-default] peer 10.1.1.1 as-number 65001
[RouterB-bgp-default] address-family ipv4 unicast
[RouterB-bgp-default-ipv4] peer 10.1.1.1 enable
[RouterB-bgp-default-ipv4] quit
[RouterB-bgp-default] quit

#配置RouterC
[RouterC] bgp 65003
[RouterC-bgp-default] router-id 3.3.3.3
[RouterC-bgp-default] confederation id 200
[RouterC-bgp-default] confederation peer-as 65001 65002
[RouterC-bgp-default] peer 10.1.2.1 as-number 65001
[RouterC-bgp-default] address-family ipv4 unicast
[RouterC-bgp-default-ipv4] peer 10.1.2.1 enable
[RouterC-bgp-default-ipv4] quit
[RouterC-bgp-default] quit


(3)配置AS 65001内的OSPF
#配置RouterA
[RouterA]ospf 1
[RouterA-ospf-1] area 0.0.0.0
[RouterA-ospf-1-area-0.0.0.0]  network 1.1.1.1 0.0.0.0
[RouterA-ospf-1-area-0.0.0.0]  network 10.1.1.0 0.0.0.255
[RouterA-ospf-1-area-0.0.0.0]  network 10.1.2.0 0.0.0.255
[RouterA-ospf-1-area-0.0.0.0]  network 10.1.3.0 0.0.0.255
[RouterA-ospf-1-area-0.0.0.0]  network 10.1.4.0 0.0.0.255

#配置RouterD
[RouterD]ospf 1
[RouterD-ospf-1] area 0.0.0.0
[RouterD-ospf-1-area-0.0.0.0]  network 4.4.4.4 0.0.0.0
[RouterD-ospf-1-area-0.0.0.0]  network 10.1.3.0 0.0.0.255
[RouterD-ospf-1-area-0.0.0.0]  network 10.1.5.0 0.0.0.255

#配置RouterE
[RouterE]ospf 1
[RouterE-ospf-1] area 0.0.0.0
[RouterE-ospf-1-area-0.0.0.0]  network 5.5.5.5 0.0.0.0
[RouterE-ospf-1-area-0.0.0.0]  network 10.1.4.0 0.0.0.255
[RouterE-ospf-1-area-0.0.0.0]  network 10.1.5.0 0.0.0.255

(4)配置AS 65001内的IBGP连接
#配置RouterA
[RouterA]bgp 65001
[RouterA-bgp-default] peer 4.4.4.4 as-number 65001
[RouterA-bgp-default] peer 4.4.4.4 connect-interface LoopBack0
[RouterA-bgp-default] peer 5.5.5.5 as-number 65001
[RouterA-bgp-default] peer 5.5.5.5 connect-interface LoopBack0
[RouterA-bgp-default] address-family ipv4 unicast
[RouterA-bgp-default-ipv4]  peer 4.4.4.4 enable
[RouterA-bgp-default-ipv4]  peer 4.4.4.4 next-hop-local
[RouterA-bgp-default-ipv4]  peer 5.5.5.5 enable
[RouterA-bgp-default-ipv4]  peer 5.5.5.5 next-hop-local

#配置RouterD
[RouterD]bgp 65001
[RouterD-bgp-default] confederation id 200
[RouterD-bgp-default] router-id 4.4.4.4
[RouterD-bgp-default] peer 1.1.1.1 as-number 65001
[RouterD-bgp-default] peer 1.1.1.1 connect-interface LoopBack0
[RouterD-bgp-default] peer 5.5.5.5 as-number 65001
[RouterD-bgp-default] peer 5.5.5.5 connect-interface LoopBack0
[RouterD-bgp-default] address-family ipv4 unicast
[RouterD-bgp-default-ipv4]  peer 1.1.1.1 enable
[RouterD-bgp-default-ipv4]  peer 1.1.1.1 next-hop-local
[RouterD-bgp-default-ipv4]  peer 5.5.5.5 enable
[RouterD-bgp-default-ipv4]  peer 5.5.5.5 next-hop-local

#配置RouterE
[RouterE]bgp 65001
[RouterE-bgp-default] confederation id 200
[RouterE-bgp-default] router-id 5.5.5.5
[RouterE-bgp-default] peer 1.1.1.1 as-number 65001
[RouterE-bgp-default] peer 1.1.1.1 connect-interface LoopBack0
[RouterE-bgp-default] peer 4.4.4.4 as-number 65001
[RouterE-bgp-default] peer 4.4.4.4 connect-interface LoopBack0
[RouterE-bgp-default] address-family ipv4 unicast
[RouterE-bgp-default-ipv4]  peer 1.1.1.1 enable
[RouterE-bgp-default-ipv4]  peer 1.1.1.1 next-hop-local
[RouterE-bgp-default-ipv4]  peer 4.4.4.4 enable
[RouterE-bgp-default-ipv4]  peer 4.4.4.4 next-hop-local


(4)配置AS 100和AS 200之间的EBGP连接
#配置RouterA
[RouterA] bgp 65001
[RouterA-bgp-default] peer 200.1.1.2 as-number 100
[RouterA-bgp-default] address-family ipv4 unicast
[RouterA-bgp-default-ipv4] peer 200.1.1.2 enable
[RouterA-bgp-default-ipv4] quit
[RouterA-bgp-default] quit

#配置RouterF
[RouterF] bgp 100
[RouterF-bgp-default] router-id 6.6.6.6 
[RouterF-bgp-default] peer 200.1.1.1 as-number 200
[RouterF-bgp-default] address-family ipv4 unicast
[RouterF-bgp-default-ipv4] peer 200.1.1.1 enable
[RouterF-bgp-default-ipv4] network 9.1.1.0 255.255.255.0
[RouterF-bgp-default-ipv4] quit
[RouterF-bgp-default] quit
```

### 4.验证配置

```bash
#查看RouterB的BGP路由表。RouterC的BGP路由表与此类似。
[RouterB]display bgp routing-table ipv4
...

     Network            NextHop         MED        LocPrf     PrefVal Path/Ogn

* >i 9.1.1.0/24         10.1.1.1        0          100        0       (65001) 100i

[RouterB]display bgp routing-table ipv4 9.1.1.0

 BGP local router ID: 2.2.2.2
 Local AS number: 65002

 Paths:   1 available, 1 best

 BGP routing table information of 9.1.1.0/24:
 From            : 10.1.1.1 (1.1.1.1)
 Rely nexthop    : 10.1.1.1
 Original nexthop: 10.1.1.1
 OutLabel        : NULL
 AS-path         : (65001) 100
 Origin          : igp
...

#查看RouterD的BGP路由表。
[RouterD]display bgp routing-table ipv4
...

     Network            NextHop         MED        LocPrf     PrefVal Path/Ogn

* >i 9.1.1.0/24         1.1.1.1         0          100        0       100i

[RouterD]display  bgp routing-table ipv4 9.1.1.0

 BGP local router ID: 4.4.4.4
 Local AS number: 65001

 Paths:   1 available, 1 best

 BGP routing table information of 9.1.1.0/24:
 From            : 1.1.1.1 (1.1.1.1)
 Rely nexthop    : 10.1.3.1
 Original nexthop: 1.1.1.1
 OutLabel        : NULL
 AS-path         : 100
 Origin          : igp
...

通过以上显示信息可以看出：
RouterF只需要和RouterA建立EBGP连接，而不需要和RouterB、RouterC建立连接，
同样可以通过联盟将路由信息传递给RouterB和RouterC。
RouterB和RouterD在同一个联盟里，但是属于不同的子自治系统，它们都是通过Router A来获取外部路由信息，
生成的BGP路由表项也是一致的，等效于在同一个自治系统内，但是又不需要物理上全连接。
```





