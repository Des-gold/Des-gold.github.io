## HTTPS配置

#### 简介

> HTTPS 是在 HTTP 与 TCP 之间加入 SSL/TLS 加密层和身份认证的通信方式，默认端口 443。客户端用服务器证书中的公钥加密对称密钥，后续数据全部用该对称密钥加密传输，中间人无法解密或篡改。
> 证书签发流程：生成根证书私钥 ca.key → 生成根证书请求 ca.csr → 自签得到根证书 ca.crt；再生成服务器私钥 server.key → 服务器请求 server.csr → 用 ca.crt 签发得到 server.crt。

### 1.安装软件

```bash
[root@server01 ~]# dnf install httpd -y
[root@server01 ~]# dnf install mod_ssl -y
[root@server01 ~]# dnf install openssl -y
```

### 2.创建CA根证书

```bash
(1)创建CA目录和索引文件
[root@server01 ~]# mkdir /etc/pki/CA
[root@server01 ~]# mkdir /etc/pki/CA/newcerts
[root@server01 ~]# cd /etc/pki/CA/
[root@server01 CA]# touch index.txt
[root@server01 CA]# echo 00 > serial

# 注:index.txt 记录已签发的证书，serial 是下一个证书的序列号，缺一个 openssl ca 就报错

(2)生成CA私钥
[root@server01 CA]# openssl genrsa -out ca.key 2048

(3)生成CA证书请求，Common Name 填CA自己的域名
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

(4)自签生成CA证书，有效期3650天
[root@server01 CA]# openssl x509 -req -days 3650 -in ca.csr -signkey ca.key -out ca.crt
```

### 3.修改openssl配置

```bash
[root@server01 CA]# vi /etc/pki/tls/openssl.cnf

# 原内容
[ v3_req ]

# Extensions to add to a certificate request

basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment

# 改为
[ v3_req ]
subjectAltName = @alt_names

# Extensions to add to a certificate request

basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
[alt_names]
DNS.1=www.6bit.icu
DNS.2=ns.6bit.icu

# 注:浏览器只认 subjectAltName，不看 Common Name，不加这段证书能签发但访问必然告警
```

### 4.创建服务器证书

```bash
(1)生成服务器私钥
[root@server01 CA]# openssl genrsa -out server.key 2048

(2)生成服务器证书请求，Common Name 填网站域名
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

(3)用CA证书签发，有效期365天
[root@server01 CA]# openssl ca -in server.csr -out server.crt -cert ca.crt -keyfile ca.key -extensions v3_req -days 365 -config /etc/pki/tls/openssl.cnf
```

### 5.修改ssl.conf

```bash
[root@server01 ~]# vi /etc/httpd/conf.d/ssl.conf

# 原内容
SSLCertificateFile /etc/pki/tls/certs/localhost.crt
SSLCertificateKeyFile /etc/pki/tls/private/localhost.key

# 改为
SSLCertificateFile /etc/pki/CA/server.crt
SSLCertificateKeyFile /etc/pki/CA/server.key
```

### 6.启动服务与放行

```bash
[root@server01 ~]# systemctl enable --now httpd
[root@server01 ~]# firewall-cmd --permanent --add-service=http
[root@server01 ~]# firewall-cmd --permanent --add-service=https
[root@server01 ~]# firewall-cmd --reload
[root@server01 ~]# setenforce 0

# 注:setenforce 0 只是临时关闭，重启失效，长期关闭改 /etc/selinux/config 的 SELINUX=disabled
```

### 7.验证

```bash
[root@server01 ~]# openssl x509 -noout -subject -issuer -dates -in /etc/pki/CA/server.crt
[root@server01 ~]# openssl s_client -connect 127.0.0.1:443 </dev/null | openssl x509 -noout -subject
```

浏览器访问 `https://www.6bit.icu`，证书是自建CA签的，客户端没装 ca.crt 照样提示不受信任，把 ca.crt 导入受信任根证书后告警消失。
