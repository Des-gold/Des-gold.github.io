# 路由控制实验

### 简介

> 互联网是由路由器连接的网络组合而成的，为了能让数据包正确地到达目标主机，路由器必须在途中进行正确地转发，这就是路由控制，路由器根据路由控制表转发数据包。

### 实验拓扑

![](https://s3.bmp.ovh/imgs/2024/11/12/9be81ef989e5aeeb.png)

### 实验需求

1、基本ip地址配置

2、R1、R2启用rip协议，并将对应接口通告到rip进程

3、R2、R3启用ospf协议，并将对应接口通告到ospf进程

4、在R2上把rip学习到的路由重分发进ospf

5、通过acl或前缀列表，把需要学习的路由匹配出来

6、R2 rip路由重分发进ospf，使用`distribute-list` 过滤路由

### 基础配置

```
Router>enable 
Router#configure terminal 
Ruijie(config)#hostname R1 
R1(config)#interface fastEthernet 0/0 
R1(config-if-FastEthernet 0/0)#ip address 192.168.1.1 255.255.255.0 
R1(config-if-FastEthernet 0/0)#exit 
R1(config)#interface loopback 0 
R1(config-if-Loopback 0)#ip address 172.16.1.1 255.255.255.224 
R1(config-if-Loopback 0)#exit 
R1(config)#interface loopback 1 
R1(config-if-Loopback 1)#ip address 172.16.1.33 255.255.255.240 
R1(config-if-Loopback 1)#exit 
R1(config)#interface loopback 2
R1(config-if-Loopback 2)#ip address 172.16.1.49 255.255.255.248 
R1(config-if-Loopback 2)#exit 
R1(config)#interface loopback 3 
R1(config-if-Loopback 3)#ip address 172.16.1.57 255.255.255.252 
R1(config-if-Loopback 3)#exit 
 
Router>enable 
Router#configure terminal 
Ruijie(config)#hostname R2 
R2(config)#interface fastEthernet 0/0
R2(config-if-FastEthernet 0/2)#ip address 192.168.1.2 255.255.255.0 
R2(config-if-FastEthernet 0/2)#exit 
R2(config)#interface fastEthernet 0/1
R2(config-if-FastEthernet 0/0)#ip address 192.168.2.1 255.255.255.0 
R2(config-if-FastEthernet 0/0)#exit 
 
Router>enable 
Router#configure terminal 
Ruijie(config)#hostname R3 
R3(config)#interface fastEthernet 0/1 
R3(config-if-FastEthernet 0/1)#ip address 192.168.2.2 255.255.255.0 
R3(config-if-FastEthernet 0/1)#exit
```

### 路由配置

```
R1(config)#router rip 
R1(config-router)#version 2     //启用rip版本2 
R1(config-router)#no auto-summary     //关闭自动汇总 
R1(config-router)#network 172.16.0.0     //将172.16.0.0的主类网络通告到rip进程 
R1(config-router)#network 192.168.1.0   
R1(config-router)#exit 
 
 
R2(config)#router rip 
R2(config-router)#version 2 
R2(config-router)#no auto-summary 
R2(config-router)#network 192.168.1.0 
R2(config-router)#exit 
R2(config)#router ospf 1    //启用ospf进程 1 
R2(config-router)#network 192.168.2.1 0.0.0.0 area 0    //将192.168.2.1对应的接口通告到ospf 进程1的区域 0 
R2(config-router)#redistribute rip subnets    //将rip路由重分发进ospf，必须加subnet 
R2(config-router)#exit 
 
R3(config)#router ospf 1 
R3(config-router)#network 192.168.2.2 0.0.0.0 area 0 
R3(config-router)#exit 
```

#### 匹配路由

1）匹配路由条目的工具有acl和前缀列表两种，只要选择其中一种就可以

2）若需要匹配一个大网段下的不同掩码长度的路由前缀时，前缀列表会更方便一些，当然用acl也是可以的，只是要写多个条目

如下示例，匹配路由条目`172.16.1.32/27`、`172.16.1.48/28`和`172.16.1.56/29`，acl需要写3个ace条目，前缀列表只要1个条目

```
1）使用acl匹配路由条目 
注意： 
此处acl匹配的是路由条目，掩码用0.0.0.0精确匹配对应的路由条目 
 
R2(config)#ip access-list standard 1 
R2(config-std-nacl)#10 permit 172.16.1.32 0.0.0.0 
R2(config-std-nacl)#20 permit 172.16.1.48 0.0.0.0 
R2(config-std-nacl)#30 permit 172.16.1.56 0.0.0.0 
R2(config-std-nacl)#exit                  
 
2）使用前缀列表匹配路由条目 
注意： 
1）前缀列表只能用来匹配路由条目，不能用来做数据包过滤 
2）前缀列表匹配的是一个网段下的子网，ge代表大于等于多少位掩码，le代表小于多少位掩码 
3）前缀列表也是从上往下匹配，最后隐含一条deny any 
 
R2(config)#ip prefix-list ruijie seq 10 permit 172.16.1.0/24 ge 28 le 30   //定义前缀列表ruijie，匹配前缀为172.16.1.0/24，子网掩码大于等于28小于等于30的路由条目 
```

#### 过滤路由

```
1）distribute-list 调用acl做路由过滤 
R2(config)#router ospf 1    
R2(config-router)#distribute-list 1 out rip     //把rip路由重分发进ospf时做路由过滤（注意方向必须是out） 
R2(config-router)#exit 
 
2）distribute-list 调用前缀列表做路由过滤 
R2(config)#router ospf 1 
R2(config-router)#distribute-list prefix ruijie out rip    //把rip路由重分发进ospf时做路由过滤（注意方向必须是out） 
R2(config-router)#exit 
```

验证

```
R3#show ip route 
...
 
Gateway of last resort is no set 
O E2 172.16.1.32/28 [110/20] via 192.168.2.1, 00:02:45, FastEthernet 0/1 
O E2 172.16.1.48/29 [110/20] via 192.168.2.1, 00:02:29, FastEthernet 0/1 
O E2 172.16.1.56/30 [110/20] via 192.168.2.1, 00:02:21, FastEthernet 0/1 
C    192.168.2.0/24 is directly connected, FastEthernet 0/1 
C    192.168.2.2/32 is local host. 
```