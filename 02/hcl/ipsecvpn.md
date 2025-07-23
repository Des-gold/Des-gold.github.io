## IKE主模式配置

#### 简介

> IKE（Internet Key  Exchange，互联网密钥交换）协议利用ISAKMP（Internet Security Association and Key Management Protocol，互联网安全联盟和密钥管理协议）语言定义密钥交换的过程，是一种对安全服务进行协商的手段。

### 1.组网图

![](https://kim-1259075102.cos.ap-guangzhou.myqcloud.com/HCL-Picture/ipsec_man.png)

| 设备     | 接口      | IP地址             | 设备    | 接口  | IP地址        |
| -------- | :-------- | :----------------- | :------ | :---- | :------------ |
| DeviceA  | GE0/0     | 202.1.1.2/24       | DeviceB | GE0/0 | 202.1.2.2/24  |
|          | GE0/1     | 10.1.1.254/24      |         | GE0/1 | 10.1.2.254/24 |
| Internet | GE0/0     | 202.1.1.1/24       |         |       |               |
|          | GE0/1     | 202.1.2.1/24       |         |       |               |
|          | Loopback0 | 114.114.114.114/24 |         |       |               |

### 2.组网需求

在Device  A和Device  B之间建立一个IPsec隧道，对Host A所在的子网（10.1.1.0/24）与Host B所在的子网（10.1.2.0/24）之间的数据流进行安全保护。具体需要求如下：

- Device A和Device B之间采用IKE协商方式建立IPsec SA。
- 使用缺省的IKE提议。
- 使用缺省的预共享密钥认证方法。
- 使用NAPT能够访问Internet

### 3.配置步骤

```bash
(1)配置配置各接口的IP地址(略)
(2)配置DeviceA
#创建高级ACL3000，定义要保护由子网10.1.1.0/24去子网10.1.2.0/24的数据流
[DeviceA]acl advanced 3000
[DeviceA-acl-ipv4-adv-3000] rule 0 permit ip source 10.1.1.0 0.0.0.255 destination 10.1.2.0 0.0.0.255

#创建IPsec安全提议tran1
[DeviceA]ipsec transform-set tran1
#配置ESP协议采用的加密算法为128比特的AES，认证算法为SHA1
[DeviceA-ipsec-transform-set-tran1] esp encryption-algorithm aes-cbc-128
[DeviceA-ipsec-transform-set-tran1] esp authentication-algorithm sha1

#创建IKE keychain，名称为keychain1
[DeviceA]ike keychain keychain1
#配置与IP地址为202.1.2.2的对端使用的预共享密钥为明文123
[DeviceA-ike-keychain-keychain1] pre-shared-key address 202.1.2.2 255.255.255.25
5 key simple 123

#创建IKE profile，名称为profile1
[DeviceA]ike profile profile1
#指定引用的IKE keychain为keychain1
[DeviceA-ike-profile-profile1] keychain keychain1
#配置本端的身份信息为IP地址202.1.1.2
[DeviceA-ike-profile-profile1] local-identity address 202.1.1.2
#配置匹配对端身份的规则为IP地址202.1.2.2
[DeviceA-ike-profile-profile1] match remote identity address 202.1.2.2 255.255.2
55.255

#创建一条IKE协商方式的IPsec安全策略，名称为map1，顺序号为10
[DeviceA]ipsec policy map1 10 isakmp
#指定引用的安全提议为tran1
[DeviceA-ipsec-policy-isakmp-map1-10] transform-set tran1
#指定引用ACL 3000
[DeviceA-ipsec-policy-isakmp-map1-10] security acl 3000
#配置IPsec隧道的对端IP地址为202.1.2.2
[DeviceA-ipsec-policy-isakmp-map1-10] remote-address 202.1.2.2
#指定引用的IKE profile为profile1
[DeviceA-ipsec-policy-isakmp-map1-10] ike-profile profile1

#创建高级ACL 3010，定义可访问外网的子网范围
[DeviceA]acl advanced 3010
#涉及NAT穿越问题需要拒绝IPsec保护的数据流，因NAT的匹配优先级大于IPsec会导致无法与对端通信
[DeviceA-acl-ipv4-adv-3010] rule 0 deny ip source 10.1.1.0 0.0.0.255 destination
 10.1.2.0 0.0.0.255
[DeviceA-acl-ipv4-adv-3010] rule 5 permit ip

#接口应用IPsec安全策略及NAT
[DeviceA]interface GigabitEthernet0/0
[DeviceA-GigabitEthernet0/0] ipsec apply policy map1
[DeviceA-GigabitEthernet0/0] nat outbound 3010

#配置默认路由
[DeviceA] ip route-static 0.0.0.0 0 202.1.1.1

(3)配置DeviceB
#创建高级ACL3000，定义要保护由子网10.1.2.0/24去子网10.1.1.0/24的数据流
[DeviceB]acl advanced 3000
[DeviceB-acl-ipv4-adv-3000] rule 0 permit ip source 10.1.2.0 0.0.0.255 destination 10.1.1.0 0.0.0.255

#创建IPsec安全提议tran1
[DeviceB]ipsec transform-set tran1
#配置ESP协议采用的加密算法为128比特的AES，认证算法为SHA1
[DeviceB-ipsec-transform-set-tran1] esp encryption-algorithm aes-cbc-128
[DeviceB-ipsec-transform-set-tran1] esp authentication-algorithm sha1

#创建IKE keychain，名称为keychain1
[DeviceB]ike keychain keychain1
#配置与IP地址为202.1.1.2的对端使用的预共享密钥为明文123
[DeviceB-ike-keychain-keychain1] pre-shared-key address 202.1.1.2 255.255.255.25
5 key simple 123

#创建IKE profile，名称为profile1
[DeviceB]ike profile profile1
#指定引用的IKE keychain为keychain1
[DeviceB-ike-profile-profile1] keychain keychain1
#配置本端的身份信息为IP地址202.1.2.2
[DeviceB-ike-profile-profile1] local-identity address 202.1.2.2
#配置匹配对端身份的规则为IP地址202.1.1.2
[DeviceB-ike-profile-profile1] match remote identity address 202.1.1.2 255.255.2
55.255

#创建一条IKE协商方式的IPsec安全策略，名称为map1，顺序号为10
[DeviceB]ipsec policy map1 10 isakmp
#指定引用的安全提议为tran1
[DeviceB-ipsec-policy-isakmp-map1-10] transform-set tran1
#指定引用ACL 3000
[DeviceB-ipsec-policy-isakmp-map1-10] security acl 3000
#配置IPsec隧道的对端IP地址为202.1.1.2
[DeviceB-ipsec-policy-isakmp-map1-10] remote-address 202.1.1.2
#指定引用的IKE profile为profile1
[DeviceB-ipsec-policy-isakmp-map1-10] ike-profile profile1

#创建高级ACL 3010，定义可访问外网的子网范围
[DeviceB]acl advanced 3010
#涉及NAT穿越问题需要拒绝IPsec保护的数据流，因NAT的匹配优先级大于IPsec会导致无法与对端通信
[DeviceB-acl-ipv4-adv-3010] rule 0 deny ip source 10.1.2.0 0.0.0.255 destination 10.1.1.0 0.0.0.255
[DeviceB-acl-ipv4-adv-3010] rule 5 permit ip

#接口应用IPsec安全策略及NAT
[DeviceB]interface GigabitEthernet0/0
[DeviceB-GigabitEthernet0/0] ipsec apply policy map1
[DeviceB-GigabitEthernet0/0] nat outbound 3010

#配置默认路由
[DeviceB] ip route-static 0.0.0.0 0 202.1.2.1 
```

### 4.验证配置

```bash
以上配置完成后，Device A和Device B之间如果有子网10.1.1.0/24与子网10.1.2.0/24之间的报文通过，将触发IKE协商。
#HostA ping HostB
[H3C]ping -a 10.1.1.1 10.1.2.1
Ping 10.1.2.1 (10.1.2.1) from 10.1.1.1: 56 data bytes, press CTRL_C to break
Request time out
56 bytes from 10.1.2.1: icmp_seq=1 ttl=253 time=0.000 ms
56 bytes from 10.1.2.1: icmp_seq=2 ttl=253 time=0.000 ms
56 bytes from 10.1.2.1: icmp_seq=3 ttl=253 time=1.000 ms
56 bytes from 10.1.2.1: icmp_seq=4 ttl=253 time=0.000 ms

#可通过如下显示信息查看到Device A和Device B上的IKE提议。因为没有配置任何IKE提议，则只显示缺省的IKE提议。
[DeviceA]display ike proposal
 Priority Authentication Authentication Encryption  Diffie-Hellman Duration
              method       algorithm    algorithm       group      (seconds)
----------------------------------------------------------------------------
 default  PRE-SHARED-KEY     SHA1       DES-CBC     Group 1        86400

[DeviceB]display ike proposal
 Priority Authentication Authentication Encryption  Diffie-Hellman Duration
              method       algorithm    algorithm       group      (seconds)
----------------------------------------------------------------------------
 default  PRE-SHARED-KEY     SHA1       DES-CBC     Group 1        86400

#查看Device A上IKE第一阶段协商成功后生成的IKE SA。
[DeviceA]display ike sa
    Connection-ID   Local               Remote              Flag      DOI
-------------------------------------------------------------------------
    1               202.1.1.2           202.1.2.2           RD        IPsec
Flags:
RD--READY RL--REPLACED FD-FADING RK-REKEY

#查看Device A上IKE第二阶段协商生成的IPsec SA。
[DeviceA]display ipsec  sa
-------------------------------
Interface: GigabitEthernet0/0
-------------------------------

  -----------------------------
  IPsec policy: map1
  Sequence number: 10
  Mode: ISAKMP
  -----------------------------
    Tunnel id: 0
    Encapsulation mode: tunnel
    Perfect Forward Secrecy:
    Inside VPN:
    Extended Sequence Numbers enable: N
    Traffic Flow Confidentiality enable: N
    Transmitting entity: Initiator
    Path MTU: 1428
    Tunnel:
        local  address: 202.1.1.2
        remote address: 202.1.2.2
    Flow:
        sour addr: 10.1.1.0/255.255.255.0  port: 0  protocol: ip
        dest addr: 10.1.2.0/255.255.255.0  port: 0  protocol: ip

    [Inbound ESP SAs]
      SPI: 3175951544 (0xbd4d2cb8)
      Connection ID: 4294967296
      Transform set: ESP-ENCRYPT-AES-CBC-128 ESP-AUTH-SHA1
      SA duration (kilobytes/sec): 1843200/3600
      SA remaining duration (kilobytes/sec): 1843199/3419
      Max received sequence-number: 4
      Anti-replay check enable: Y
      Anti-replay window size: 64
      UDP encapsulation used for NAT traversal: N
      Status: Active

    [Outbound ESP SAs]
      SPI: 931156075 (0x3780506b)
      Connection ID: 4294967297
      Transform set: ESP-ENCRYPT-AES-CBC-128 ESP-AUTH-SHA1
      SA duration (kilobytes/sec): 1843200/3600
      SA remaining duration (kilobytes/sec): 1843199/3419
      Max sent sequence-number: 4
      UDP encapsulation used for NAT traversal: N
      Status: Active
      
#HostA ping Internet
[H3C]ping -a 10.1.1.1 114.114.114.114
Ping 114.114.114.114 (114.114.114.114) from 10.1.1.1: 56 data bytes, press CTRL_C to break
56 bytes from 114.114.114.114: icmp_seq=0 ttl=254 time=1.000 ms
56 bytes from 114.114.114.114: icmp_seq=1 ttl=254 time=0.000 ms
56 bytes from 114.114.114.114: icmp_seq=2 ttl=254 time=0.000 ms
56 bytes from 114.114.114.114: icmp_seq=3 ttl=254 time=1.000 ms
56 bytes from 114.114.114.114: icmp_seq=4 ttl=254 time=0.000 ms
```





##  IKE野蛮模式配置

### 1.组网图

![](https://kim-1259075102.cos.ap-guangzhou.myqcloud.com/HCL-Picture/ipsec_aggressive.png)

| 设备    | 接口  | IP地址        | 设备     | 接口      | IP地址             |
| ------- | :---- | :------------ | :------- | :-------- | :----------------- |
| DeviceA | GE0/0 | 202.1.1.2/24  | DeviceB  | GE0/0     | DHCP               |
|         | GE0/1 | 10.1.1.254/24 |          | GE0/1     | 10.1.2.254/24      |
| DeviceC | GE0/0 | DHCP          | Internet | GE0/0     | 202.1.1.1/24       |
|         | GE0/1 | 10.1.3.254/24 |          | GE0/1     | 202.1.2.1/24       |
|         |       |               |          | GE0/2     | 202.1.3.1/24       |
|         |       |               |          | Loopback0 | 114.114.114.114/24 |

### 2.组网需求

在DeviceA与DeviceB和DeviceC之间建立一个IPsec隧道，对Host A所在的子网（10.1.1.0/24）与Host B所在的子网（10.1.2.0/24）和HostC所在的子网（10.1.3.0/24）之间的数据流进行安全保护。具体需要求如下：

- Internet作为DHCP server，DeviceB与DeviceC公网接口均为自动获取地址

- 协商双方使用缺省的IKE提议。
- 协商模式为野蛮模式协商。
- 第一阶段协商的认证方法为预共享密钥认证。

### 3.配置步骤

```bash
(1)配置配置各接口的IP地址(略)
(2)配置Internet
#配置DHCP Server
[Internet]dhcp enable
[Internet]dhcp server ip-pool 1
[Internet-dhcp-pool-1] gateway-list 202.1.2.1
[Internet-dhcp-pool-1] network 202.1.2.0 mask 255.255.255.0
[Internet-dhcp-pool-1]quit
[Internet]dhcp server ip-pool 2
[Internet-dhcp-pool-2] gateway-list 202.1.3.1
[Internet-dhcp-pool-2] network 202.1.3.0 mask 255.255.255.0
[Internet-dhcp-pool-2]quit
[Internet]interface GigabitEthernet0/1
[Internet-GigabitEthernet0/1] dhcp server apply ip-pool 1
[Internet-GigabitEthernet0/1]quit
[Internet]interface GigabitEthernet0/2
[Internet-GigabitEthernet0/2] dhcp server apply ip-pool 2

#配置DeviceB接口为自动获取
[DeviceB]interface GigabitEthernet0/0
[DeviceB-GigabitEthernet0/0] ip address dhcp-alloc

#配置DeviceC接口为自动获取
[DeviceB]interface GigabitEthernet0/0
[DeviceB-GigabitEthernet0/0] ip address dhcp-alloc

(3)配置DeviceA
#配置本端身份为FQDN名称DeviceA
[DeviceA] ike identity fqdn DeviceA

#创建IPsec安全提议tran1
[DeviceA] ipsec transform-set tran1
#配置ESP协议采用的加密算法为128比特的AES，认证算法为SHA1
[DeviceA-ipsec-transform-set-tran1] esp encryption-algorithm 3des-cbc
[DeviceA-ipsec-transform-set-tran1] esp authentication-algorithm sha1

#创建IKE keychain，名称为keychain1
[DeviceA]ike keychain keychain1
#配置与FQDN为DeviceB的对端使用的预共享密钥为明文123
[DeviceA-ike-keychain-keychain1] pre-shared-key hostname DeviceB key simple 123

#创建IKE keychain，名称为keychain2
[DeviceA]ike keychain keychain2
#配置与FQDN为DeviceC的对端使用的预共享密钥为明文123
[DeviceA-ike-keychain-keychain2] pre-shared-key hostname DeviceC key simple 123

#创建IKE profile，名称为profile1
[DeviceA] ike profile profile1
#指定引用的IKE keychain为keychain1
[DeviceA-ike-profile-profile1] keychain keychain1
#配置协商模式为野蛮模式。
[DeviceA-ike-profile-profile1] exchange-mode aggressive
#配置匹配对端身份为FQDN名称DeviceB
[DeviceA-ike-profile-profile1] match remote identity fqdn DeviceB

#创建IKE profile，名称为profile2
[DeviceA]ike profile profile2
#指定引用的IKE keychain为keychain2
[DeviceA-ike-profile-profile2] keychain keychain2
#配置协商模式为野蛮模式。
[DeviceA-ike-profile-profile2] exchange-mode aggressive
#配置匹配对端身份为FQDN名称DeviceC
[DeviceA-ike-profile-profile2] match remote identity fqdn DeviceC

#创建IPsec安全策略模板，名称为dymap，顺序号为1
[DeviceA]ipsec policy-template dymap 1
[DeviceA-ipsec-policy-template-dymap-1] transform-set tran1
[DeviceA-ipsec-policy-template-dymap-1] ike-profile profile1
#创建IPsec安全策略模板，名称为dymap，顺序号为2
[DeviceA]ipsec policy-template dymap 2
[DeviceA-ipsec-policy-template-dymap-2] transform-set tran1
[DeviceA-ipsec-policy-template-dymap-2] ike-profile profile2

#创建IPsec安全策略，名称为map与IPsec安全策略模板dymap进行映射
[DeviceA]ipsec policy map 10 isakmp template dymap

#创建高级ACL 3010，定义可访问外网的子网范围
[DeviceA]acl advanced 3010
[DeviceA-acl-ipv4-adv-3010] rule 0 permit ip

#接口应用IPsec安全策略及NAT
[DeviceA]interface GigabitEthernet0/0
[DeviceA-GigabitEthernet0/0] ipsec apply policy map
[DeviceA-GigabitEthernet0/0] nat outbound 3010

#配置默认路由
[DeviceA]ip route-static 0.0.0.0 0 202.1.1.1

(3)配置DeviceB
#创建高级ACL3000，定义要保护由子网10.1.2.0/24去子网10.1.1.0/24的数据流
[DeviceB]acl advanced 3000
[DeviceB-acl-ipv4-adv-3000] rule 0 permit ip source 10.1.2.0 0.0.0.255 destination 10.1.1.0 0.0.0.255

#配置本端身份为FQDN名称DeviceB
[DeviceB] ike identity fqdn DeviceB

#创建IPsec安全提议tran1
[DeviceB]ipsec transform-set tran1
#配置ESP协议采用的加密算法为128比特的AES，认证算法为SHA1
[DeviceB-ipsec-transform-set-tran1] esp encryption-algorithm aes-cbc-128
[DeviceB-ipsec-transform-set-tran1] esp authentication-algorithm sha1

#创建IKE keychain，名称为keychain1
[DeviceB]ike keychain keychain1
#配置与IP地址为202.1.1.2的对端使用的预共享密钥为明文123
[DeviceB-ike-keychain-keychain1] pre-shared-key address 202.1.1.2 255.255.255.25
5 key simple 123

#创建IKE profile，名称为profile1
[DeviceB] ike profile profile1
#指定引用的IKE keychain为keychain1
[DeviceB-ike-profile-profile1] keychain keychain1
#配置协商模式为野蛮模式。
[DeviceB-ike-profile-profile1] exchange-mode aggressive
#配置匹配对端身份为FQDN名称DeviceB
[DeviceB-ike-profile-profile1] match remote identity fqdn DeviceA

#创建一条IKE协商方式的IPsec安全策略，名称为map1，顺序号为10
[DeviceB]ipsec policy map1 10 isakmp
#指定引用的安全提议为tran1
[DeviceB-ipsec-policy-isakmp-map1-10] transform-set tran1
#指定引用ACL 3000
[DeviceB-ipsec-policy-isakmp-map1-10] security acl 3000
#配置IPsec隧道的对端IP地址为202.1.1.2
[DeviceB-ipsec-policy-isakmp-map1-10] remote-address 202.1.1.2
#指定引用的IKE profile为profile1
[DeviceB-ipsec-policy-isakmp-map1-10] ike-profile profile1

#创建高级ACL 3010，定义可访问外网的子网范围
[DeviceB]acl advanced 3010
#涉及NAT穿越问题需要拒绝IPsec保护的数据流，因NAT的匹配优先级大于IPsec会导致无法与对端通信
[DeviceB-acl-ipv4-adv-3010] rule 0 deny ip source 10.1.2.0 0.0.0.255 destination 10.1.1.0 0.0.0.255
[DeviceB-acl-ipv4-adv-3010] rule 5 permit ip

#接口应用IPsec安全策略及NAT
[DeviceB]interface GigabitEthernet0/0
[DeviceB-GigabitEthernet0/0] ipsec apply policy map1
[DeviceB-GigabitEthernet0/0] nat outbound 3010

(4)配置DeviceC
#创建高级ACL3000，定义要保护由子网10.1.3.0/24去子网10.1.1.0/24的数据流
[DeviceC]acl advanced 3000
[DeviceC-acl-ipv4-adv-3000] rule 0 permit ip source 10.1.3.0 0.0.0.255 destination 10.1.1.0 0.0.0.255

#配置本端身份为FQDN名称DeviceC
[DeviceC] ike identity fqdn DeviceC

#创建IPsec安全提议tran1
[DeviceC]ipsec transform-set tran1
#配置ESP协议采用的加密算法为128比特的AES，认证算法为SHA1
[DeviceC-ipsec-transform-set-tran1] esp encryption-algorithm aes-cbc-128
[DeviceC-ipsec-transform-set-tran1] esp authentication-algorithm sha1

#创建IKE keychain，名称为keychain1
[DeviceC]ike keychain keychain1
#配置与IP地址为202.1.1.2的对端使用的预共享密钥为明文123
[DeviceC-ike-keychain-keychain1] pre-shared-key address 202.1.1.2 255.255.255.25
5 key simple 123

#创建IKE profile，名称为profile1
[DeviceC] ike profile profile1
#指定引用的IKE keychain为keychain1
[DeviceC-ike-profile-profile1] keychain keychain1
#配置协商模式为野蛮模式。
[DeviceC-ike-profile-profile1] exchange-mode aggressive
#配置匹配对端身份为FQDN名称DeviceC
[DeviceC-ike-profile-profile1] match remote identity fqdn DeviceA

#创建一条IKE协商方式的IPsec安全策略，名称为map1，顺序号为10
[DeviceC]ipsec policy map1 10 isakmp
#指定引用的安全提议为tran1
[DeviceC-ipsec-policy-isakmp-map1-10] transform-set tran1
#指定引用ACL 3000
[DeviceC-ipsec-policy-isakmp-map1-10] security acl 3000
#配置IPsec隧道的对端IP地址为202.1.1.2
[DeviceC-ipsec-policy-isakmp-map1-10] remote-address 202.1.1.2
#指定引用的IKE profile为profile1
[DeviceC-ipsec-policy-isakmp-map1-10] ike-profile profile1

#创建高级ACL 3010，定义可访问外网的子网范围
[DeviceC]acl advanced 3010
#涉及NAT穿越问题需要拒绝IPsec保护的数据流，因NAT的匹配优先级大于IPsec会导致无法与对端通信
[DeviceC-acl-ipv4-adv-3010] rule 0 deny ip source 10.1.3.0 0.0.0.255 destination 10.1.1.0 0.0.0.255
[DeviceC-acl-ipv4-adv-3010] rule 5 permit ip

#接口应用IPsec安全策略及NAT
[DeviceC]interface GigabitEthernet0/0
[DeviceC-GigabitEthernet0/0] ipsec apply policy map1
[DeviceC-GigabitEthernet0/0] nat outbound 3010
```

### 4.验证配置

```bash
以上配置完成后，子网10.1.2.0/24和10.1.3.0/24若向子网10.1.1.0/24发送报文，将触发IKE协商。
#HostB ping HostA
[H3C]ping -a 10.1.2.1 10.1.1.1
Ping 10.1.1.1 (10.1.1.1) from 10.1.2.1: 56 data bytes, press CTRL_C to break
56 bytes from 10.1.1.1: icmp_seq=0 ttl=253 time=1.000 ms
...

#HostC ping HostA
[H3C]ping -a 10.1.3.1 10.1.1.1
Ping 10.1.1.1 (10.1.1.1) from 10.1.3.1: 56 data bytes, press CTRL_C to break
56 bytes from 10.1.1.1: icmp_seq=0 ttl=253 time=1.000 ms
...



#查看到DeviceB上IKE第一阶段协商成功后生成的IKE SA
[DeviceB]display ike sa
    Connection-ID   Local               Remote              Flag      DOI
-------------------------------------------------------------------------
    8               202.1.2.2           202.1.1.2           RD        IPsec
Flags:
RD--READY RL--REPLACED FD-FADING RK-REKEY

[DeviceB]display ike sa   verbose
   -----------------------------------------------
   Connection ID: 8
   Outside VPN:
   Inside VPN:
   Profile: profile1
   Transmitting entity: Initiator
   Initiator cookie: a822b1ed1dcacd0a
   Responder cookie: 447b244b63910dd4
   -----------------------------------------------
   Local IP/port: 202.1.2.2/500
   Local ID type: FQDN
   Local ID: DeviceB

   Remote IP/port: 202.1.1.2/500
   Remote ID type: FQDN
   Remote ID: DeviceA

   Authentication-method: PRE-SHARED-KEY
   Authentication-algorithm: SHA1
   Encryption-algorithm: DES-CBC

   Life duration(sec): 86400
   Remaining key duration(sec): 84068
   Exchange-mode: Aggressive
   Diffie-Hellman group: Group 1
   NAT traversal: Not detected

   Extend authentication: Disabled
   Assigned IP address:
   Vendor ID index:0xffffffff
   Vendor ID sequence number:0x0

#查看到DeviceC上IKE第一阶段协商成功后生成的IKE SA
[DeviceC]display  ike sa
    Connection-ID   Local               Remote              Flag      DOI
-------------------------------------------------------------------------
    1               202.1.3.2           202.1.1.2           RD        IPsec
Flags:
RD--READY RL--REPLACED FD-FADING RK-REKEY

[DeviceC]display ike sa  verbose
   -----------------------------------------------
   Connection ID: 1
   Outside VPN:
   Inside VPN:
   Profile: profile1
   Transmitting entity: Initiator
   Initiator cookie: 0f3b1dce957df305
   Responder cookie: 37f25c1bca199861
   -----------------------------------------------
   Local IP/port: 202.1.3.2/500
   Local ID type: FQDN
   Local ID: DeviceC

   Remote IP/port: 202.1.1.2/500
   Remote ID type: FQDN
   Remote ID: DeviceA

   Authentication-method: PRE-SHARED-KEY
   Authentication-algorithm: SHA1
   Encryption-algorithm: DES-CBC

   Life duration(sec): 86400
   Remaining key duration(sec): 84114
   Exchange-mode: Aggressive
   Diffie-Hellman group: Group 1
   NAT traversal: Not detected

   Extend authentication: Disabled
   Assigned IP address:
   Vendor ID index:0xffffffff
   Vendor ID sequence number:0x0

由上方显示信息（Exchange-mode: Aggressive）得知IKE协商模式为野蛮模式
```

