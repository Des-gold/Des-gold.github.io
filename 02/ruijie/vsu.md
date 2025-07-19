# VSU实验

### 简介

> VSU（Virtual Switching Unit，虚拟交换单元）是一种网络设备多虚一（N:1）技术，通过将多台网络设备虚拟成一台逻辑设备管理和使用，以简化运维设备和网络拓扑。同时外围设备可以通过聚合链路连接到VSU系统中的不同成员设备，实现跨设备链路冗余，以提升网络可靠性和扩展性。

### 实验需求

1.S1和S2间的Gi0/48端口作为双主机检测链路，配置基于BFD的双主机检

2.主设备：Domain id：1,switch id:1,priority 150;备设备：Domain id：1,switch id:2,priority 120

### 配置

```
Ruijie>enable  
Ruijie#configure terminal 
Ruijie(config)#hostname S1    
S1(config)#switch virtual domain  1 	
S1(config-vs-domain)#switch  1 
S1(config-vs-domain)#switch  1  priority 150 
S1(config-vs-domain)#exit 
S1(config)#vsl-port 			//VSL链路至少需要2条，一条链路可靠性较低，当出现链路震荡时，VSU会非常不稳定
S1(config-vsl-port)#port-member  interface  tenGigabitEthernet 0/25
S1config-vsl-port)#port-member  interface  tenGigabitEthernet 0/26
S1(config-vs-domain)#end
S1#switch  convert mode  virtual 		//转换为VSU模式
Convert mode will backup and delete config file, and reload the switch. Are you sure to continue[yes/no]:yes
 
 
Ruijie(config)#host S2
S2(config)#switch virtual domain  1 		//domaind id 必须和第一台一致 
S2(config-vs-domain)#switch  2 
S2(config-vs-domain)#switch  2 priority  120 
S2(config)#vsl-port 
S2(config-vsl-port)#port-member  interface  tenGigabitEthernet 0/25
S2(config-vsl-port)#port-member  interface  tenGigabitEthernet 0/26
S2(config-vsl-port)#end
S2#switch  convert  mode  virtual 
Convert mode will backup and delete config file, and reload the switch. Are you sure to continue[yes/no]:yes
 
选择转换模式后，设备会重启启动，并组建VSU。
```

### 基于BFD检测

BFD的双主机检测端口必须是**三层路由口**，二层口、三层AP口或SVI口都不能作为BFD的检测端口；

当用户将双主机检测端口从三层路由口转换为其他类型的端口模式时，BFD的双主机检测配置将自动清除，并给出提示。BFD检测采用扩展BFD，不能通过现有BFD的配置与显示命令配置双主机检测口。

```
S1(config)#int gi 1/0/24 
S1(config-if-GigabitEthernet 1/0/24)#no switchport 
S1(config-if-GigabitEthernet 1/0/24)#int gi 2/0/24 
S1(config-if-GigabitEthernet 2/0/24)#no switchpor
S1(config)#switch virtual  domain  1 
S1(config-vs-domain)#dual-active detection bfd 
S1(config-vs-domain)#dual-active  bfd interface  gi1/0/24
S1(config-vs-domain)#dual-active  bfd interface  gi2/0/24
```

#### 验证

```
Ruijie(config)#show switch virtual 
Switch_id     Domain_id     Priority     Position     Status     Role          Description                      
----------------------------------------------------------------------------------------------------------------
1(1)          1(1)          150(150)     LOCAL        OK         ACTIVE        Access-Switch-Virtual-Switch1    
2(2)          1(1)          120(120)     REMOTE       OK         STANDBY       Access-Switch-Virtual-Switch1 
 
ACTIVE 表示为主机箱
STANDBY 表示为备机箱
```