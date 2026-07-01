## 二层注册配置

#### 简介

> 本文档介绍AP与AC间通过二层网络完成注册的配置举例。

### 1.组网图

<img src="https://kim-1259075102.cos.ap-guangzhou.myqcloud.com/HCL-Picture/acregister_l2.png" />

| 设备      | 接口    | IP地址          | 设备 | 接口    | IP地址          |
| --------- | ------- | --------------- | ---- | ------- | --------------- |
| L3_swtich | Vlan100 | 192.168.10.1/24 | AC   | Vlan100 | 192.168.10.2/24 |
|           | Vlan200 | 192.168.20.1/24 |      |         |                 |

### 2.组网需求

集中式转发架构下，AC旁挂在L2 switch1上，L3 switch做DHCP server为AP、Client和Host分配IP地址。需要实现无线客户端Client通过AP连接到AC上，具体要求如下：

- 无线客户端Client通过VLAN 200接入网络；
- AC属于VLAN 100，AC和AP之间通过二层网络建立连接。

### 3.配置步骤

```bash
(1)配置各接口的类型
#配置L3_switch
[L3_switch]vlan 100
[L3_switch-vlan100]vlan 200
[L3_switch]interface GigabitEthernet1/0/1
[L3_switch-GigabitEthernet1/0/1] port link-type trunk
[L3_switch-GigabitEthernet1/0/1] port trunk permit vlan 1 100 200

#配置L2_switch
[L2_switch]vlan 100
[L2_switch-vlan100]vlan 200
[L2_switch]interface GigabitEthernet1/0/1
[L2_switch-GigabitEthernet1/0/1] port link-type trunk
[L2_switch-GigabitEthernet1/0/1] port trunk permit vlan  100 200
[L2_switch]interface GigabitEthernet1/0/2
[L2_switch-GigabitEthernet1/0/2] port link-type trunk
[L2_switch-GigabitEthernet1/0/2] port trunk permit vlan  100 200
[L2_switch]interface GigabitEthernet1/0/3
[L2_switch-GigabitEthernet1/0/3] port access vlan 100

#配置AC
[AC]vlan 100
[AC-vlan100]vlan 200
[AC]interface GigabitEthernet1/0/0
[AC-GigabitEthernet1/0/0] port link-type trunk
[AC-GigabitEthernet1/0/0] port trunk permit vlan  100 200

(2)配置各接口的IP地址
#配置L3_switch
[L3_switch]dhcp enable
[L3_switch]dhcp server ip-pool vlan100
[L3_switch-dhcp-pool-vlan100] gateway-list 192.168.10.1
[L3_switch-dhcp-pool-vlan100] network 192.168.10.0 mask 255.255.255.0
[L3_switch-dhcp-pool-vlan100]quit
[L3_switch]dhcp server ip-pool vlan200
[L3_switch-dhcp-pool-vlan200] gateway-list 192.168.20.1
[L3_switch-dhcp-pool-vlan200] network 192.168.20.0 mask 255.255.255.0
[L3_switch-dhcp-pool-vlan200]quit
[L3_switch]interface Vlan-interface100
[L3_switch-Vlan-interface100] ip address 192.168.10.1 255.255.255.0
[L3_switch-Vlan-interface100] dhcp server apply ip-pool vlan100
[L3_switch-Vlan-interface100]quit
[L3_switch-Vlan-interface100]interface Vlan-interface200
[L3_switch-Vlan-interface200] ip address 192.168.20.1 255.255.255.0
[L3_switch-Vlan-interface200] dhcp server apply ip-pool vlan200

#配置AC
[AC]interface Vlan-interface100
[AC-Vlan-interface100] ip address 192.168.10.2 255.255.255.0
[AC]ip route-static 0.0.0.0 0.0.0.0 192.168.10.1

(3)配置AC无线控制器
#配置无线模板
[AC]wlan service-template  1
[AC-wlan-st-1]ssid kim
#配置客户端数据报文转发位置为AC(隧道转发/集中转发)。
[AC-wlan-st-1]client forwarding-location ac
[AC-wlan-st-1]service-template enable

#配置AP上线
[AC]wlan ap AP1 model WA6320-HCL
#配置AP的序列号为H3C_A8-89-D5-6A-05-00
[AC-wlan-ap-AP1] serial-id H3C_A8-89-D5-6A-05-00
# 创建AP组group1，并配置AP名称入组规则
[AC]wlan ap-group group1
[AC-wlan-ap-group-group1]ap AP1
Please wait.....Done.
# 将无线服务模板1绑定到AP组group1下的Radio 1上，并指定客户端上线的VLAN为VLAN 200
[AC-wlan-ap-group-group1]ap-model WA6320-HCL
[AC-wlan-ap-group-group1-ap-model-WA6320-HCL]radio 1
[AC-wlan-ap-group-group1-ap-model-WA6320-HCL-radio-1]service-template 1 vlan 200
[AC-wlan-ap-group-group1-ap-model-WA6320-HCL-radio-1]radio enable

补充：
#查看AP版本
[H3C]display  device
Slot No.   Board Type          Status      Software Version
-----------------------------------------------------------
0          WA6320-HCL          Master      COMWAREV700R001
#查看AP序列号
<H3C>display  device manuinfo
DEVICE_ID:Slot ID:0
DEVICE_NAME:Simware
DEVICE_SERIAL_NUMBER:H3C_a8-89-d5-6a-05-00(序列号)  
MAC_ADDRESS:a8-89-d5-6a-05-04
MANUFACTURING_DATE:2014-7-16
VENDOR_NAME:H3C
Subslot 1:
The operation is not supported on the specified slot or subslot.
...

#配置AC自动注册
#开启自动AP功能。
[AC] wlan auto-ap enable
#创建一个AP管理模板，其名称为ap-group，型号名称为WA6320-HCL
[AC] wlan ap ap-group model WA6320-HCL
#设置自动查找AP序列号。
[AC-wlan-ap-ap-group] serial-id auto
[AC-wlan-ap-ap-group] quit
#将所有自动AP转换为固化AP，但名称保持不变。
[AC] wlan auto-ap persistent all
```

### 4.验证配置

```bash
#查看AP状态
[AC]display  wlan ap all
...

                                 AP information
 State : I = Idle,      J  = Join,       JA = JoinAck,    IL = ImageLoad
         C = Config,    DC = DataCheck,  R  = Run,   M = Master,  B = Backup

AP name                        APID  State Model           Serial ID
AP1                            1     R/M   WA6320-HCL      H3C_A8-89-D5-6A-05-00

#查看AP客户端
[AC]display wlan client
Total number of clients: 1

MAC address    User name            AP name               R IP address      VLAN
00e0-0602-1235 N/A                  AP1                   1 192.168.20.2    200

```







## 三层注册配置

#### 简介

> 本文档介绍AP与AC间通过二层网络完成注册的配置举例。

### 1.组网图

<img src="https://kim-1259075102.cos.ap-guangzhou.myqcloud.com/HCL-Picture/acregister_l3.png" />

| 设备      | 接口    | IP地址          | 设备 | 接口    | IP地址          |
| --------- | ------- | --------------- | ---- | ------- | --------------- |
| L3_swtich | Vlan100 | 192.168.10.1/24 | AC   | Vlan100 | 192.168.10.2/24 |
|           | Vlan200 | 192.168.20.1/24 |      |         |                 |
|           | Vlan300 | 192.168.30.1/24 |      |         |                 |

### 2.组网需求

集中式转发架构下，AC旁挂在L3 switch上，L3 switch做DHCP server为AP、Client和Host分配IP地址。需要实现无线客户端Client通过AP连接到AC上，并能与有线客户端Host互相访问，具体要求如下：

- 无线客户端Client通过VLAN 300接入网络；
- AC属于VLAN 100，AP属于VLAN 200，AC和AP之间跨三层网络建立连

### 3.配置步骤

```bash
(1)配置各接口的类型
#配置L3_switch
[L3_switch]vlan 100
[L3_switch-vlan100]vlan 200
[L3_switch-vlan200]vlan 300
[L3_switch]interface GigabitEthernet1/0/1
[L3_switch-GigabitEthernet1/0/1] port link-type trunk
[L3_switch-GigabitEthernet1/0/1] port trunk permit vlan 100 200 300
[L3_switch]interface GigabitEthernet1/0/2
[L3_switch-GigabitEthernet1/0/2] port link-type trunk
[L3_switch-GigabitEthernet1/0/2] port trunk permit vlan 100 200 300

#配置L2_switch
[L2_switch]vlan 100
[L2_switch-vlan100]vlan 200
[L2_switch-vlan200]vlan 300
[L2_switch]interface GigabitEthernet1/0/1
[L2_switch-GigabitEthernet1/0/1] port link-type trunk
[L2_switch-GigabitEthernet1/0/1] port trunk permit vlan  100 200 300 
[L2_switch]interface GigabitEthernet1/0/2
[L2_switch-GigabitEthernet1/0/2] port access vlan 200

#配置AC
[AC]vlan 100
[AC-vlan100]vlan 200
[AC-vlan200]vlan 300
[AC]interface GigabitEthernet1/0/0
[AC-GigabitEthernet1/0/0] port link-type trunk
[AC-GigabitEthernet1/0/0] port trunk permit vlan  100 200 300 

(2)配置各接口的IP地址
#配置L3_switch
[L3_switch]interface Vlan-interface100
[L3_switch-Vlan-interface100] ip address 192.168.10.1 255.255.255.0
[L3_switch-Vlan-interface100] dhcp server apply ip-pool vlan100
[L3_switch-Vlan-interface100]quit
[L3_switch-Vlan-interface100]interface Vlan-interface200
[L3_switch-Vlan-interface200] ip address 192.168.20.1 255.255.255.0
[L3_switch-Vlan-interface200] dhcp server apply ip-pool vlan200
[L3_switch-Vlan-interface200] quit
[L3_switch]interface Vlan-interface300
[L3_switch-Vlan-interface300] ip address 192.168.30.1 255.255.255.0
[L3_switch-Vlan-interface300] dhcp server apply ip-pool vlan300

#配置AC
[AC]interface Vlan-interface100
[AC-Vlan-interface100] ip address 192.168.10.2 255.255.255.0
[AC]ip route-static 0.0.0.0 0.0.0.0 192.168.10.1

(3)配置DHCP server
#配置L3_switch
[L3_switch]dhcp enable
[L3_switch]dhcp server ip-pool vlan100
[L3_switch-dhcp-pool-vlan100] gateway-list 192.168.10.1
[L3_switch-dhcp-pool-vlan100] network 192.168.10.0 mask 255.255.255.0
[L3_switch-dhcp-pool-vlan100]quit
[L3_switch]dhcp server ip-pool vlan200
[L3_switch-dhcp-pool-vlan200] gateway-list 192.168.20.1
[L3_switch-dhcp-pool-vlan200] network 192.168.20.0 mask 255.255.255.0
#配置DHCP Option43的内容为AC的十六进制IP地址，其中8007000001为固定搭配。
[L3_switch-dhcp-pool-vlan200] option 43 hex 8007000001c0a80a02
[L3_switch]dhcp server ip-pool vlan300
[L3_switch-dhcp-pool-vlan300] gateway-list 192.168.30.1
[L3_switch-dhcp-pool-vlan300] network 192.168.30.0 mask 255.255.255.0
[L3_switch]interface Vlan-interface100
[L3_switch-Vlan-interface100] dhcp server apply ip-pool vlan100
[L3_switch-Vlan-interface100]quit
[L3_switch-Vlan-interface100]interface Vlan-interface200
[L3_switch-Vlan-interface200] dhcp server apply ip-pool vlan200
[L3_switch-Vlan-interface200] quit
[L3_switch]interface Vlan-interface300
[L3_switch-Vlan-interface300] dhcp server apply ip-pool vlan300

(4)配置AC无线控制器
#配置无线模板
[AC]wlan service-template  1
[AC-wlan-st-1]ssid kim
#配置客户端数据报文转发位置为AC(隧道转发/集中转发)。
[AC-wlan-st-1]client forwarding-location ac
[AC-wlan-st-1]service-template enable

#配置AP上线
[AC]wlan ap AP1 model WA6320-HCL
#配置AP的序列号为H3C_A8-89-D5-6A-05-00
[AC-wlan-ap-AP1] serial-id H3C_A8-89-D5-6A-05-00
# 创建AP组group1，并配置AP名称入组规则
[AC]wlan ap-group group1
[AC-wlan-ap-group-group1]ap AP1
Please wait.....Done.
# 将无线服务模板1绑定到AP组group1下的Radio 1上，并指定客户端上线的VLAN为VLAN 300
[AC-wlan-ap-group-group1]ap-model WA6320-HCL
[AC-wlan-ap-group-group1-ap-model-WA6320-HCL]radio 1
[AC-wlan-ap-group-group1-ap-model-WA6320-HCL-radio-1]service-template 1 vlan 300
[AC-wlan-ap-group-group1-ap-model-WA6320-HCL-radio-1]radio enable

补充：
#查看AP版本
[H3C]display  device
Slot No.   Board Type          Status      Software Version
-----------------------------------------------------------
0          WA6320-HCL          Master      COMWAREV700R001
#查看AP序列号
<H3C>display  device manuinfo
DEVICE_ID:Slot ID:0
DEVICE_NAME:Simware
DEVICE_SERIAL_NUMBER:H3C_a8-89-d5-6a-05-00(序列号)  
MAC_ADDRESS:a8-89-d5-6a-05-04
MANUFACTURING_DATE:2014-7-16
VENDOR_NAME:H3C
Subslot 1:
The operation is not supported on the specified slot or subslot.
...
```

### 4.验证配置

```bash
#查看AP状态
[AC]display  wlan ap all
...

                                 AP information
 State : I = Idle,      J  = Join,       JA = JoinAck,    IL = ImageLoad
         C = Config,    DC = DataCheck,  R  = Run,   M = Master,  B = Backup

AP name                        APID  State Model           Serial ID
AP1                            1     R/M   WA6320-HCL      H3C_A8-89-D5-6A-05-00

#查看AP客户端
[AC]display wlan client
Total number of clients: 1

MAC address    User name            AP name               R IP address      VLAN
00e0-0402-1235 N/A                  AP1                   1 192.168.30.2    300
```

