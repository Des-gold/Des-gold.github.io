# HTTPS配置

### 简介

> HTTPS （全称：Hypertext Transfer Protocol Secure），是以安全为目标的 HTTP 通道，在HTTP的基础上通过传输加密和身份认证保证了传输过程的安全性 [1]。HTTPS 在HTTP 的基础下加入SSL，HTTPS 的安全基础是 SSL，因此加密的详细内容就需要 SSL。 HTTPS 存在不同于 HTTP 的默认端口及一个加密/身份验证层（在 HTTP与 TCP 之间）。这个系统提供了身份验证与加密通讯方法。它被广泛用于万维网上安全敏感的通讯，例如交易支付等方面。

生成私钥（.key）–>生成证书请求（.csr）–>用CA根证书签名得到证书（.crt）

### 安装需要的软件

```
[root@server01 ~]# dnf install httpd -y
[root@server01 ~]# dnf install mod_ssl –y
[root@server01 ~]# dnf install openssl –y
```

### 创建CA根证书

#### 创建需要的文件

```
[root@server01 ~]# mkdir /etc/pki/CA
[root@server01 ~]# mkdir /etc/pki/CA/newcerts
[root@server01 ~]# cd /etc/pki/CA/
[root@server01 CA]# touch index.txt
[root@server01 CA]# echo 00 > serial
```

#### 创建CA证书的私钥

```
[root@server01 CA]# openssl genrsa -out ca.key 2048
```

#### 创建CA证书请求

```
[root@server01 CA]# openssl req -new -key ca.key -out ca.csr
 
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:CN
State or Province Name (full name) []:GZ
Locality Name (eg, city) [Default City]:KL
Organization Name (eg, company) [Default Company Ltd]:6BIT
Organizational Unit Name (eg, section) []:IT
Common Name (eg, your name or your server's hostname) []:ns.6bit.icu
Email Address []:
 
Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
```

#### 生成CA证书

```
[root@server01 CA]# openssl x509 -req -days 3650 -in ca.csr -signkey ca.key -out ca.crt
```

### 更改openssl配置文件

```
[root@server01 CA]# vi /etc/pki/tls/openssl.cnf
 
====查询下方内容====
 
[ v3_req ]
 
# Extensions to add to a certificate request
 
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
 
====添加下方内容====
 
[ v3_req ]
subjectAltName = @alt_names
# Extensions to add to a certificate request
 
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
[alt_names]
DNS.1=www.6bit.icu
DNS.2=ns.6bit.icu
```

### 创建服务器证书

#### 生成服务器私钥

```
[root@server01 CA]# openssl genrsa -out server.key 2048
```

#### 生成服务器请求证书

```
[root@server01 CA]# openssl req -new -key server.key -out server.csr -config /etc/pki/tls/openssl.cnf
 
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:CN
State or Province Name (full name) []:GZ
Locality Name (eg, city) [Default City]:KL
Organization Name (eg, company) [Default Company Ltd]:6BIT
Organizational Unit Name (eg, section) []:IT
Common Name (eg, your name or your server's hostname) []:www.6bit.icu
Email Address []:
 
Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
```

#### 生成服务器证书

```
[root@server01 CA]# openssl ca -in server.csr -out server.crt -cert ca.crt -keyfile ca.key -extensions v3_req -days 365 -config /etc/pki/tls/openssl.cnf
```

### 更改SSL配置文件

```
[root@server01 ~]# vi /etc/httpd/conf.d/ssl.conf
 
====找下面两行====
 
SSLCertificateFile /etc/pki/tls/certs/localhost.crt
 
SSLCertificateKeyFile /etc/pki/tls/private/localhost.key
 
====更改路径====
 
SSLCertificateFile /etc/pki/CA/server.crt
 
SSLCertificateKeyFile /etc/pki/CA/server.key
```

### 防火墙放行

```
[root@server01 ~]# systemctl enable --now httpd
[root@server01 ~]# firewall-cmd --permanent --add-service=http
[root@server01 ~]# firewall-cmd --permanent --add-service=https
[root@server01 ~]# firewall-cmd –reload
[root@server01 ~]# setenforce 0
```