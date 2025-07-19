## M-LAG基本功能配置

#### 简介

> M-LAG（Multichassis link aggregation，跨设备链路聚合）将两台物理设备在聚合层面虚拟成一台设备来实现跨设备链路聚合，从而提供设备级冗余保护和流量负载分担。

### 1.组网图

![](https://kim-1259075102.cos.ap-guangzhou.myqcloud.com/HCL-Picture/m-lag_basic.png)

注：系统版本为HCL v5.5.0，设备型号为S6850

### 2.组网需求

- 由于用户对于业务的可靠性要求很高，如果Device C和接入设备（Device A和Device B）之间配置链路聚合能保证链路级的可靠性，接入设备发生故障时则会导致业务中断。这时用户可以采用M-LAG技术，正常工作时链路进行负载分担且任何一台设备故障对业务均没有影响，保证业务的高可靠性。

-  配置三层以太网接口为保留接口，在该三层以太网接口上搭建Keepalive链路，保证Keepalive报文能够正常传输。

### 3.配置步骤

```bash
(1)配置Device A
# 系统配置
<DeviceA> system-view
[DeviceA] m-lag system-mac 1-1-1
[DeviceA] m-lag system-number 1
[DeviceA] m-lag system-priority 123

# 配置Keepalive报文的目的IP地址和源IP地址。
[DeviceA] m-lag keepalive ip destination 1.1.1.1 source 1.1.1.2

# 配置端口GigabitEthernet1/0/5工作在三层模式，并配置IP地址为Keepalive报文的源IP地址。
[DeviceA] interface GigabitEthernet 1/0/5
[DeviceA-GigabitEthernet1/0/5] port link-mode route
[DeviceA-GigabitEthernet1/0/5] ip address 1.1.1.2 24
[DeviceA-GigabitEthernet1/0/5] quit

# 配置Keepalive链路接口为保留接口。
[DeviceA] m-lag mad exclude interface  GigabitEthernet 1/0/5

# 创建二层聚合接口3，并配置该接口为动态聚合模式。
[DeviceA] interface bridge-aggregation 3
[DeviceA-Bridge-Aggregation3] link-aggregation mode dynamic
[DeviceA-Bridge-Aggregation3] quit

# 分别将端口GigabitEthernet1/0/1和GigabitEthernet1/0/2加入到聚合组3中。
[DeviceA] interface GigabitEthernet 1/0/1
[DeviceA-GigabitEthernet1/0/1] port link-aggregation group 3
[DeviceA-GigabitEthernet1/0/1] quit
[DeviceA] interface GigabitEthernet 1/0/2
[DeviceA-GigabitEthernet1/0/2] port link-aggregation group 3
[DeviceA-GigabitEthernet1/0/2] quit

# 将二层聚合接口3配置为peer-link接口。
[DeviceA] interface bridge-aggregation 3
[DeviceA-Bridge-Aggregation3] port m-lag peer-link 1
[DeviceA-Bridge-Aggregation3] undo mac-address static source-check enable
[DeviceA-Bridge-Aggregation3] quit

# 创建二层聚合接口4，并配置该接口为动态聚合模式。
[DeviceA] interface bridge-aggregation 4
[DeviceA-Bridge-Aggregation4] link-aggregation mode dynamic
[DeviceA-Bridge-Aggregation4] quit

# 分别将端口GigabitEthernet1/0/3和GigabitEthernet1/0/4加入到聚合组4中。
[DeviceA] interface GigabitEthernet 1/0/3
[DeviceA-GigabitEthernet1/0/3] port link-aggregation group 4
[DeviceA-GigabitEthernet1/0/3] quit
[DeviceA] interface GigabitEthernet 1/0/4
[DeviceA-GigabitEthernet1/0/4] port link-aggregation group 4
[DeviceA-GigabitEthernet1/0/4] quit

# 将二层聚合接口4加入M-LAG组4中。
[DeviceA] interface bridge-aggregation 4
[DeviceA-Bridge-Aggregation4] port m-lag group 4
[DeviceA-Bridge-Aggregation4] quit


(2)配置DeviceB
# 系统配置
<DeviceB> system-view
[DeviceB] m-lag system-mac 1-1-1
[DeviceB] m-lag system-number 2
[DeviceB] m-lag system-priority 123

# 配置Keepalive报文的目的IP地址和源IP地址。
[DeviceB] m-lag keepalive ip destination 1.1.1.2 source 1.1.1.1

# 配置端口GigabitEthernet1/0/5工作在三层模式，并配置IP地址为Keepalive报文的源IP地址。
[DeviceB] interface GigabitEthernet 1/0/5
[DeviceB-GigabitEthernet1/0/5] port link-mode route
[DeviceB-GigabitEthernet1/0/5] ip address 1.1.1.1 24
[DeviceB-GigabitEthernet1/0/5] quit

# 配置Keepalive链路接口为保留接口。
[DeviceB] m-lag mad exclude interface GigabitEthernet 1/0/5

# 创建二层聚合接口3，并配置该接口为动态聚合模式。
[DeviceB] interface bridge-aggregation 3
[DeviceB-Bridge-Aggregation3] link-aggregation mode dynamic
[DeviceB-Bridge-Aggregation3] quit

# 分别将端口GigabitEthernet1/0/1和GigabitEthernet1/0/2加入到聚合组3中。
[DeviceB] interface GigabitEthernet 1/0/1
[DeviceB-GigabitEthernet1/0/1] port link-aggregation group 3
[DeviceB-GigabitEthernet1/0/1] quit
[DeviceB] interface GigabitEthernet 1/0/2
[DeviceB-GigabitEthernet1/0/2] port link-aggregation group 3
[DeviceB-GigabitEthernet1/0/2] quit

# 将二层聚合接口3配置为peer-link接口。
[DeviceB] interface bridge-aggregation 3
[DeviceB-Bridge-Aggregation3] port m-lag peer-link 1
[DeviceB-Bridge-Aggregation3] undo mac-address static source-check enable
[DeviceB-Bridge-Aggregation3] quit

# 创建二层聚合接口4，并配置该接口为动态聚合模式。
[DeviceB] interface bridge-aggregation 4
[DeviceB-Bridge-Aggregation4] link-aggregation mode dynamic
[DeviceB-Bridge-Aggregation4] quit

# 分别将端口GigabitEthernet1/0/3和GigabitEthernet1/0/4加入到聚合组4中。
[DeviceB] interface GigabitEthernet 1/0/3
[DeviceB-GigabitEthernet1/0/3] port link-aggregation group 4
[DeviceB-GigabitEthernet1/0/3] quit
[DeviceB] interface GigabitEthernet 1/0/4
[DeviceB-GigabitEthernet1/0/4] port link-aggregation group 4
[DeviceB-GigabitEthernet1/0/4] quit

# 将二层聚合接口4加入M-LAG组4中。
[DeviceB] interface bridge-aggregation 4
[DeviceB-Bridge-Aggregation4] port m-lag group 4
[DeviceB-Bridge-Aggregation4] quit


(3)配置Device C
# 创建二层聚合接口4，并配置该接口为动态聚合模式。
<DeviceC> system-view
[DeviceC] interface bridge-aggregation 4
[DeviceC-Bridge-Aggregation4] link-aggregation mode dynamic
[DeviceC-Bridge-Aggregation4] quit

# 分别将端口GigabitEthernet1/0/1～GigabitEthernet1/0/4加入到聚合组4中。
[DeviceC] interface range GigabitEthernet 1/0/1 to GigabitEthernet 1/0/4
[DeviceC-if-range] port link-aggregation group 4
[DeviceC-if-range] quit

```

### 4.验证配置

```bash
# 查看Device A上M-LAG的Keepalive报文信息。
[DeviceA]display  m-lag keepalive
Neighbor keepalive link status (cause): Up
Neighbor is alive for: 58 s 852 ms
Keepalive packet transmission status:
  Sent: Successful
  Received: Successful
Last received keepalive packet information:
  Source IP address: 1.1.1.1
  Time: 2025/07/19 16:55:18
  Action: Accept

M-LAG keepalive parameters:
Destination IP address: 1.1.1.1
Source IP address: 1.1.1.2
Keepalive UDP port : 6400
Keepalive VPN name : N/A
Keepalive interval : 1000 ms
Keepalive timeout  : 5 sec
Keepalive hold time: 3 sec

# 查看Device B上M-LAG的Keepalive报文信息。
[DeviceB]display  m-lag keepalive
Neighbor keepalive link status (cause): Up
Neighbor is alive for: 117 s 937 ms
Keepalive packet transmission status:
  Sent: Successful
  Received: Successful
Last received keepalive packet information:
  Source IP address: 1.1.1.2
  Time: 2025/07/19 16:56:18
  Action: Accept

M-LAG keepalive parameters:
Destination IP address: 1.1.1.2
Source IP address: 1.1.1.1
Keepalive UDP port : 6400
Keepalive VPN name : N/A
Keepalive interval : 1000 ms
Keepalive timeout  : 5 sec
Keepalive hold time: 3 sec
以上信息表明Device A和Device B设备间无故障。

# 查看Device C上聚合组4的详细信息。
[DeviceC]display  link-aggregation  verbose   Bridge-Aggregation  4
...
Aggregate Interface: Bridge-Aggregation4
Creation Mode: Manual
Aggregation Mode: Dynamic
Loadsharing Type: Shar
Management VLANs: None
System ID: 0x8000, 7e2b-7c29-0300
Local:
  Port                Status   Priority Index    Oper-Key               Flag
  GE1/0/1             S        32768    1        1                      {ACDEF}
  GE1/0/2             S        32768    2        1                      {ACDEF}
  GE1/0/3             S        32768    3        1                      {ACDEF}
  GE1/0/4             S        32768    4        1                      {ACDEF}
Remote:
  Actor               Priority Index    Oper-Key SystemID               Flag
  GE1/0/1(R)          32768    16386    40004    0x7b  , 0001-0001-0001 {ACDEF}
  GE1/0/2             32768    16388    40004    0x7b  , 0001-0001-0001 {ACDEF}
  GE1/0/3             32768    32770    40004    0x7b  , 0001-0001-0001 {ACDEF}
  GE1/0/4             32768    32772    40004    0x7b  , 0001-0001-0001 {ACDEF}
以上信息表明，Device C的端口GigabitEthernet1/0/1～GigabitEthernet1/0/4均处于选中状态(ACDEF)，
此时Device C将DeviceA和DeviceB认为是一台设备，从而实现了跨设备的聚合。
```



## M-LAG三层转发配置举例

### 1.组网图

![](https://kim-1259075102.cos.ap-guangzhou.myqcloud.com/HCL-Picture/m-lag_l3.png)

注：系统版本为HCL v5.5.0，设备型号为S6850

### 2.组网需求

- 由于用户对于业务的可靠性要求很高，如果Device C和接入设备（Device A和Device B）之间配置链路聚合只能保证链路级的可靠性，接入设备发生故障时则会导致业务中断。这时用户可以采用M-LAG技术，正常工作时链路进行负载分担且任何一台设备故障对业务均没有影响，保证业务的高可靠性。

-  配置Device A和Device B的三层以太网接口GigabitEthernet1/0/5为保留接口，在该三层以太网接口上搭建Keepalive链路，保证Keepalive报文能够正常传输。

- VLAN 100内主机的缺省网关为10.1.1.100/24，VLAN 101内主机的缺省网关为20.1.1.100/24。Device A和Device B同时属于虚拟IP地址为10.1.1.100/24的备份组1和虚拟IP地址为20.1.1.100/24的备份组2。在备份组1和备份组2中Device A的优先级高于Device B。

### 3.配置步骤

```bash
(1)配置DeviceA
# 系统配置。
<DeviceA> system-view
[DeviceA] m-lag system-mac 1-1-1
[DeviceA] m-lag system-number 1
[DeviceA] m-lag system-priority 123

# 配置Keepalive报文的目的IP地址和源IP地址。
[DeviceA] m-lag keepalive ip destination 1.1.1.2 source 1.1.1.1

# 配置端口GigabitEthernet1/0/5工作在三层模式，并配置IP地址为Keepalive报文的源IP地址。
[DeviceA] interface GigabitEthernet 1/0/5
[DeviceA-GigabitEthernet1/0/5] port link-mode route
[DeviceA-GigabitEthernet1/0/5] ip address 1.1.1.1 24
[DeviceA-GigabitEthernet1/0/5] quit

# 配置Keepalive链路接口为M-LAG保留接口。
[DeviceA] m-lag mad exclude interface GigabitEthernet 1/0/5

# 创建动态二层聚合接口125。
[DeviceA] interface bridge-aggregation 125
[DeviceA-Bridge-Aggregation125] link-aggregation mode dynamic
[DeviceA-Bridge-Aggregation125] quit

# 分别将端口GigabitEthernet1/0/3和GigabitEthernet1/0/4加入到聚合组125中。
[DeviceA] interface GigabitEthernet 1/0/3
[DeviceA-GigabitEthernet1/0/3] port link-aggregation group 125
[DeviceA-GigabitEthernet1/0/3] quit
[DeviceA] interface GigabitEthernet 1/0/4
[DeviceA-GigabitEthernet1/0/4] port link-aggregation group 125
[DeviceA-GigabitEthernet1/0/4] quit

# 配置二层聚合接口125为peer-link接口。
[DeviceA] interface bridge-aggregation 125
[DeviceA-Bridge-Aggregation125] port m-lag peer-link 1
[DeviceA-Bridge-Aggregation125] undo mac-address static source-check enable
[DeviceA-Bridge-Aggregation125] quit

# 创建动态二层聚合接口100，并配置该接口为M-LAG接口1。
[DeviceA] interface bridge-aggregation 100
[DeviceA-Bridge-Aggregation100] link-aggregation mode dynamic
[DeviceA-Bridge-Aggregation100] port m-lag group 1
[DeviceA-Bridge-Aggregation100] quit

# 将端口GigabitEthernet1/0/1加入到聚合组100中。
[DeviceA] interface GigabitEthernet 1/0/1
[DeviceA-GigabitEthernet1/0/1] port link-aggregation group 100
[DeviceA-GigabitEthernet1/0/1] quit

# 创建动态二层聚合接口101，并配置该接口为M-LAG接口2。
[DeviceA] interface bridge-aggregation 101
[DeviceA-Bridge-Aggregation101] link-aggregation mode dynamic
[DeviceA-Bridge-Aggregation101] port m-lag group 2
[DeviceA-Bridge-Aggregation101] quit

# 将端口GigabitEthernet1/0/2加入到聚合组101中。
[DeviceA] interface GigabitEthernet 1/0/2
[DeviceA-GigabitEthernet1/0/2] port link-aggregation group 101
[DeviceA-GigabitEthernet1/0/2] quit

# 创建VLAN 100和101。
[DeviceA] vlan 100
[DeviceA-vlan100] quit
[DeviceA] vlan 101
[DeviceA-vlan101] quit

# 配置二层聚合接口100为Trunk端口，并允许VLAN 100的报文通过。
[DeviceA] interface bridge-aggregation 100
[DeviceA-Bridge-Aggregation100] port link-type trunk
[DeviceA-Bridge-Aggregation100] port trunk permit vlan 100
[DeviceA-Bridge-Aggregation100] quit

# 配置二层聚合接口101为Trunk端口，并允许VLAN 101的报文通过。
[DeviceA] interface bridge-aggregation 101
[DeviceA-Bridge-Aggregation101] port link-type trunk
[DeviceA-Bridge-Aggregation101] port trunk permit vlan 101
[DeviceA-Bridge-Aggregation101] quit

# 创建接口Vlan-interface100和Vlan-interface101，并配置其IP地址。
[DeviceA] interface vlan-interface 100
[DeviceA-vlan-interface100] ip address 10.1.1.1 24
[DeviceA-vlan-interface100] quit
[DeviceA] interface vlan-interface 101
[DeviceA-vlan-interface101] ip address 20.1.1.1 24
[DeviceA-vlan-interface101] quit

# 配置Vlan-interface100和Vlan-interface101接口为M-LAG保留接口。
[DeviceA] m-lag mad exclude interface vlan-interface 100
[DeviceA] m-lag mad exclude interface vlan-interface 101

# 配置OSPF。
[DeviceA] ospf
[DeviceA-ospf-1] import-route direct
[DeviceA-ospf-1] area 0
[DeviceA-ospf-1-area-0.0.0.0] network 10.1.1.0 0.0.0.255
[DeviceA-ospf-1-area-0.0.0.0] network 20.1.1.0 0.0.0.255
[DeviceA-ospf-1-area-0.0.0.0] quit
[DeviceA-ospf-1] quit

# 为接口Vlan-interface100创建备份组1，并配置备份组1的虚拟IP地址为10.1.1.100。
[DeviceA] interface vlan-interface 100
[DeviceA-Vlan-interface100] vrrp vrid 1 virtual-ip 10.1.1.100

# 设置Device A在备份组1中的优先级为200，以保证Device A成为Master，从而和Device A在M-LAG中角色一致。
[DeviceA-Vlan-interface100] vrrp vrid 1 priority 200
[DeviceA-Vlan-interface100] quit

# 为接口Vlan-interface101创建备份组2，并配置备份组2的虚拟IP地址为20.1.1.100。
[DeviceA] interface vlan-interface 101
[DeviceA-Vlan-interface101] vrrp vrid 2 virtual-ip 20.1.1.100

# 设置Device A在备份组2中的优先级为200，以保证Device A成为Master，从而和Device A在M-LAG中角色一致。
[DeviceA-Vlan-interface101] vrrp vrid 2 priority 200
[DeviceA-Vlan-interface101] quit

(2)配置DeviceB
# 系统配置。
<DeviceB> system-view
[DeviceB] m-lag system-mac 1-1-1
[DeviceB] m-lag system-number 2
[DeviceB] m-lag system-priority 123

# 配置Keepalive报文的目的IP地址和源IP地址。
[DeviceB] m-lag keepalive ip destination 1.1.1.1 source 1.1.1.2

# 配置端口GigabitEthernet1/0/5工作在三层模式，并配置IP地址为Keepalive报文的源IP地址。
[DeviceB] interface GigabitEthernet 1/0/5
[DeviceB-GigabitEthernet1/0/5] port link-mode route
[DeviceB-GigabitEthernet1/0/5] ip address 1.1.1.2 24
[DeviceB-GigabitEthernet1/0/5] quit

# 配置Keepalive链路接口为M-LAG保留接口。
[DeviceB] m-lag mad exclude interface GigabitEthernet 1/0/5

# 创建动态二层聚合接口125。
[DeviceB] interface bridge-aggregation 125
[DeviceB-Bridge-Aggregation125] link-aggregation mode dynamic
[DeviceB-Bridge-Aggregation125] quit

# 分别将端口GigabitEthernet1/0/3和GigabitEthernet1/0/4加入到聚合组125中。
[DeviceB] interface GigabitEthernet 1/0/3
[DeviceB-GigabitEthernet1/0/3] port link-aggregation group 125
[DeviceB-GigabitEthernet1/0/3] quit
[DeviceB] interface GigabitEthernet 1/0/4
[DeviceB-GigabitEthernet1/0/4] port link-aggregation group 125
[DeviceB-GigabitEthernet1/0/4] quit

# 配置二层聚合接口125为peer-link接口。
[DeviceB] interface bridge-aggregation 125
[DeviceB-Bridge-Aggregation125] port m-lag peer-link 1
[DeviceB-Bridge-Aggregation125] undo mac-address static source-check enable
[DeviceB-Bridge-Aggregation125] quit

# 创建动态二层聚合接口100，并配置该接口为M-LAG接口1。
[DeviceB] interface bridge-aggregation 100
[DeviceB-Bridge-Aggregation100] link-aggregation mode dynamic
[DeviceB-Bridge-Aggregation100] port m-lag group 1
[DeviceB-Bridge-Aggregation100] quit

# 将端口GigabitEthernet1/0/1加入到聚合组100中。
[DeviceB] interface GigabitEthernet 1/0/1
[DeviceB-GigabitEthernet1/0/1] port link-aggregation group 100
[DeviceB-GigabitEthernet1/0/1] quit

# 创建动态二层聚合接口101，并配置该接口为M-LAG接口2。
[DeviceB] interface bridge-aggregation 101
[DeviceB-Bridge-Aggregation101] link-aggregation mode dynamic
[DeviceB-Bridge-Aggregation101] port m-lag group 2
[DeviceB-Bridge-Aggregation101] quit

# 将端口GigabitEthernet1/0/2加入到聚合组101中。
[DeviceB] interface GigabitEthernet 1/0/2
[DeviceB-GigabitEthernet1/0/2] port link-aggregation group 101
[DeviceB-GigabitEthernet1/0/2] quit

# 创建VLAN 100和101。
[DeviceB] vlan 100
[DeviceB-vlan100] quit
[DeviceB] vlan 101
[DeviceB-vlan101] quit

# 配置二层聚合接口100为Trunk端口，并允许VLAN 100的报文通过。
[DeviceB] interface bridge-aggregation 100
[DeviceB-Bridge-Aggregation100] port link-type trunk
[DeviceB-Bridge-Aggregation100] port trunk permit vlan 100
[DeviceB-Bridge-Aggregation100] quit

# 配置二层聚合接口101为Trunk端口，并允许VLAN 101的报文通过。
[DeviceB] interface bridge-aggregation 101
[DeviceB-Bridge-Aggregation101] port link-type trunk
[DeviceB-Bridge-Aggregation101] port trunk permit vlan 101
[DeviceB-Bridge-Aggregation101] quit

# 创建接口Vlan-interface100和Vlan-interface101，并配置其IP地址。
[DeviceB] interface vlan-interface 100
[DeviceB-vlan-interface100] ip address 10.1.1.2 24
[DeviceB-vlan-interface100] quit
[DeviceB] interface vlan-interface 101
[DeviceB-vlan-interface101] ip address 20.1.1.2 24
[DeviceB-vlan-interface101] quit

# 配置Vlan-interface100和Vlan-interface101接口为M-LAG保留接口。
[DeviceB] m-lag mad exclude interface vlan-interface 100
[DeviceB] m-lag mad exclude interface vlan-interface 101

# 配置OSPF。
[DeviceB] ospf
[DeviceB-ospf-1] import-route direct
[DeviceB-ospf-1] area 0
[DeviceB-ospf-1-area-0.0.0.0] network 10.1.1.0 0.0.0.255
[DeviceB-ospf-1-area-0.0.0.0] network 20.1.1.0 0.0.0.255
[DeviceB-ospf-1-area-0.0.0.0] quit
[DeviceB-ospf-1] quit

# 为接口Vlan-interface100创建备份组1，并配置备份组1的虚拟IP地址为10.1.1.100。
[DeviceB] interface vlan-interface 100
[DeviceB-Vlan-interface100] vrrp vrid 1 virtual-ip 10.1.1.100
[DeviceB-Vlan-interface100] quit

# 为接口Vlan-interface101创建备份组2，并配置备份组2的虚拟IP地址为20.1.1.100。
[DeviceB] interface vlan-interface 101
[DeviceB-Vlan-interface101] vrrp vrid 2 virtual-ip 20.1.1.100
[DeviceB-Vlan-interface101] quit

(3)配置DeviceC
# 创建二层聚合接口100，并配置该接口为动态聚合模式。
<DeviceC> system-view
[DeviceC] interface bridge-aggregation 100
[DeviceC-Bridge-Aggregation100] link-aggregation mode dynamic
[DeviceC-Bridge-Aggregation100] quit

# 分别将端口GigabitEthernet1/0/1和GigabitEthernet1/0/2加入到聚合组100中。
[DeviceC] interface range GigabitEthernet 1/0/1 to GigabitEthernet 1/0/2
[DeviceC-if-range] port link-aggregation group 100
[DeviceC-if-range] quit

# 创建VLAN 100。
[DeviceC] vlan 100
[DeviceC-vlan100] quit

# 配置二层聚合接口100为Trunk端口，并允许VLAN 100的报文通过。
[DeviceC] interface bridge-aggregation 100
[DeviceC-Bridge-Aggregation100] port link-type trunk
[DeviceC-Bridge-Aggregation100] port trunk permit vlan 100
[DeviceC-Bridge-Aggregation100] quit

# 配置接口GigabitEthernet1/0/3为Trunk端口，并允许VLAN 100的报文通过。
[DeviceC] interface GigabitEthernet 1/0/3
[DeviceC-GigabitEthernet1/0/3] port link-type trunk
[DeviceC-GigabitEthernet1/0/3] port trunk permit vlan 100
[DeviceC-GigabitEthernet1/0/3] quit

# 创建接口Vlan-interface100，并配置其IP地址。
[DeviceC] interface vlan-interface 100
[DeviceC-vlan-interface100] ip address 10.1.1.3 24
[DeviceC-vlan-interface100] quit

# 配置OSPF。
[DeviceC] ospf
[DeviceC-ospf-1] import-route direct
[DeviceC-ospf-1] area 0
[DeviceC-ospf-1-area-0.0.0.0] network 10.1.1.0 0.0.0.255
[DeviceC-ospf-1-area-0.0.0.0] quit
[DeviceC-ospf-1] quit

(4)配置DeviceD
# 创建二层聚合接口101，并配置该接口为动态聚合模式。
<DeviceD> system-view
[DeviceD] interface bridge-aggregation 101
[DeviceD-Bridge-Aggregation101] link-aggregation mode dynamic
[DeviceD-Bridge-Aggregation101] quit

# 分别将端口GigabitEthernet1/0/1和GigabitEthernet1/0/2加入到聚合组101中。
[DeviceD] interface range GigabitEthernet 1/0/1 to GigabitEthernet 1/0/2
[DeviceD-if-range] port link-aggregation group 101
[DeviceD-if-range] quit

# 创建VLAN 101。
[DeviceD] vlan 101
[DeviceD-vlan101] quit

# 配置二层聚合接口101为Trunk端口，并允许VLAN 101的报文通过。
[DeviceD] interface bridge-aggregation 101
[DeviceD-Bridge-Aggregation101] port link-type trunk
[DeviceD-Bridge-Aggregation101] port trunk permit vlan 101
[DeviceD-Bridge-Aggregation101] quit

# 配置接口GigabitEthernet1/0/3为Trunk端口，并允许VLAN 101的报文通过。
[DeviceD] interface GigabitEthernet 1/0/3
[DeviceD-GigabitEthernet1/0/3] port link-type trunk
[DeviceD-GigabitEthernet1/0/3] port trunk permit vlan 101
[DeviceD-GigabitEthernet1/0/3] quit

# 创建接口Vlan-interface101，并配置其IP地址。
[DeviceD] interface vlan-interface 101
[DeviceD-vlan-interface101] ip address 20.1.1.3 24
[DeviceD-vlan-interface101] quit

# 配置OSPF。
[DeviceD] ospf
[DeviceD-ospf-1] import-route direct
[DeviceD-ospf-1] area 0
[DeviceD-ospf-1-area-0.0.0.0] network 20.1.1.0 0.0.0.255
[DeviceD-ospf-1-area-0.0.0.0] quit
[DeviceD-ospf-1] quit
```

### 4.验证配置

```bash
# 查看Device A上M-LAG的Keepalive报文信息。
[DeviceA]display m-lag keepalive
Neighbor keepalive link status (cause): Up
Neighbor is alive for: 64 s 411 ms
Keepalive packet transmission status:
  Sent: Successful
  Received: Successful
Last received keepalive packet information:
  Source IP address: 1.1.1.1
  Time: 2025/07/19 17:47:12
  Action: Accept

M-LAG keepalive parameters:
Destination IP address: 1.1.1.1
Source IP address: 1.1.1.2
Keepalive UDP port : 6400
Keepalive VPN name : N/A
Keepalive interval : 1000 ms
Keepalive timeout  : 5 sec
Keepalive hold time: 3 sec

# 查看Device B上M-LAG的Keepalive报文信息。
[DeviceB]display m-lag keepalive
Neighbor keepalive link status (cause): Up
Neighbor is alive for: 81 s 883 ms
Keepalive packet transmission status:
  Sent: Successful
  Received: Successful
Last received keepalive packet information:
  Source IP address: 1.1.1.2
  Time: 2025/07/19 17:47:29
  Action: Accept

M-LAG keepalive parameters:
Destination IP address: 1.1.1.2
Source IP address: 1.1.1.1
Keepalive UDP port : 6400
Keepalive VPN name : N/A
Keepalive interval : 1000 ms
Keepalive timeout  : 5 sec
Keepalive hold time: 3 sec


# 在Device C和Device D上分别查看二层聚合组100和二层聚合组101的详细信息。
可以看到Device C和Device D的端口GigabitEthernet1/0/1、GigabitEthernet1/0/2均处于选中状态，
此时Device C和Device D将DeviceA和DeviceB认为是一台设备，从而实现了跨设备的聚合。

[DeviceC]display link-aggregation verbose   Bridge-Aggregation  100
...
Local:
  Port                Status   Priority Index    Oper-Key               Flag
  GE1/0/1             S        32768    1        1                      {ACDEF}
  GE1/0/2             S        32768    2        1                      {ACDEF}
Remote:
  Actor               Priority Index    Oper-Key SystemID               Flag
  GE1/0/1(R)          32768    16386    40001    0x7b  , 0001-0001-0001 {ACDEF}
  GE1/0/2             32768    32770    40001    0x7b  , 0001-0001-0001 {ACDEF}

[DeviceD]display link-aggregation verbose   Bridge-Aggregation  101
...
Local:
  Port                Status   Priority Index    Oper-Key               Flag
  GE1/0/1             S        32768    1        1                      {ACDEF}
  GE1/0/2             S        32768    2        1                      {ACDEF}
Remote:
  Actor               Priority Index    Oper-Key SystemID               Flag
  GE1/0/1(R)          32768    16387    40002    0x7b  , 0001-0001-0001 {ACDEF}
  GE1/0/2             32768    32771    40002    0x7b  , 0001-0001-0001 {ACDEF}
```







