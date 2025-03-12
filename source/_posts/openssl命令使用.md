---
title: openssl命令使用
date: 2025-03-06T00:36:18+08:00
categories: 技术文章
index_img: /images/ssl/index.webp
banner_img: /images/banner.webp
excerpt: 介绍 openssl 命令使用
tags:
  - tech
  - tools
  - linux
---

## openssl 命令模块
openssl 命令主要包含以下几个模块

{% note primary %}
1. version  查看版本信息
2. s_client/s_server 通用SSL/TLS测试工具
3. genrsa 用于生成私钥
4. x509 x509 证书管理
5. verify: x509 证书验证
6. rsa: RSA密钥管理（例如对证书签名）
7. rsautl: 用于完成RSA签名，验证，加密和解密功能
8. req: 生成证书签名请求(CSR)
9. enc: 用于加解密
10. dgst: 生成信息摘要
11. passwd: 生成散列密码
12. rand: 生成伪随机数
13. pkcs7: PSCS#7协议数据管理
14. ca: CA管理(例如对证书进行签名)
{% endnote %}
这里主要是记录一些常用命令的使用情况


## 常用命令
### s_client
#### 测试 SSL/TLS 连接
```bash
openssl s_client -connect www.baidu.com:443
```
* ***作用*** ：连接到`www.baidu.com` 的 443 端口，测试 SSL/TLS 连接。
* ***输出*** ：显示服务器的证书信息、协议版本、加密套件等。
---

#### 指定SNI
```bash
openssl s_client -connect www.baidu.com:443 -servername xxx.baidu.com
```
* ***作用*** ：在连接时候指定 SNI，确保返回的证书是匹配的
* ***场景*** ：当服务器托管多个域名时，使用 SNI 指定目标域名。
---

#### 检查证书链
```bash
openssl s_client -connect www.baiduc.com:443 -showcerts
```
* ***作用*** ：显示服务器返回的完整证书链
* ***输出*** ：返回包含的服务器证书和中间证书
---

#### 验证证书
```bash
openssl s_client -connect example.com:443 -CAfile ca.crt
```
* ***作用*** ：使用指定的 CA 证书文件验证服务器证书。
* ***场景*** ：检查服务器证书是否由受信任的 CA 签发。

#### 指定协议版本
```bash
openssl s_client -connect example.com:443 -tls1_2
```
* ***作用*** ：指定使用 TLS 1.2 协议进行连接。
* ***其他选项*** ：
    -tls1：使用 TLS 1.0。
    -tls1_1：使用 TLS 1.1。
    -tls1_3：使用 TLS 1.3。
----

#### 检查客户端证书
```bash
openssl s_client -connect example.com:443 -cert client.crt -key client.key
```
* ***作用*** ：使用客户端证书进行双向认证。
* ***场景*** ：测试服务器是否支持客户端证书验证。
---

#### 检查支持的加密套件
```bash 
openssl s_client -connect www.baidu.com:443 -cipher ECDHE-RSA-AES128-GCM-SHA256
```
* ***作用***：指定使用高强度的加密套件进行连接。
* ***场景*** ：测试服务器是否支持特定的加密套件。
---


`s_client`主要是以下是常用场景：
1. 测试连接：-connect example.com:443
2. 检查证书：-showcerts -CAfile ca.crt
3. 调试问题：-debug -status
4. 验证协议：-tls1_2
5. 测试加密套件：-cipher HIGH


### s_server
#### 启动SSL/TLS 服务器
```bash
openssl s_server -cert server.crt -key server.key -accept 4433
```
* -cert server.crt：指定服务器证书文件。
* -key server.key：指定服务器私钥文件。
* -accept 4433：监听 4433 端口。    
---
* ***作用*** ：启动一个 SSL/TLS 服务器，使用指定的证书和私钥。
* ***输出*** ：服务器监听默认端口 4433，等待客户端连接。


#### 启用客户端证书校验
```bash
openssl s_server -cert server.crt -key server.key -CAfile ca.crt -verify 1 -accept 4433
```
* -CAfile ca.crt：指定 CA 证书文件，用于验证客户端证书。
* -verify 1：启用客户端证书验证，并设置验证深度为 1。

#### 启动简单的 HTTP 服务器功能
```bash
openssl s_server -cert server.crt -key server.key -accept 4433 -HTTP
```

#### 显示详细调试信息
```bash
openssl s_server -cert server.crt -key server.key -accept 4433 -debug -state
```
* -debug：显示详细的 SSL/TLS 握手信息。
* -state：显示 SSL/TLS 状态信息。

#### 指定加密套件
```bash
openssl s_server -cert server.crt -key server.key -accept 4433 -cipher "ECDHE-RSA-AES128-GCM-SHA256"
```

#### 强制使用 tls 1.3
```bash
openssl s_server -cert server.crt -key server.key -accept 4433 -tls1_3
```



### 生成证书
使用 OpenSSL 生成证书步骤分为两种常见场景：生成自签名证书和生成由 CA 签名的证书

#### 生成自签名证书
这种适合测试或内部环境，不需要依赖第三方 CA

***步骤***
```bash
# 1. 生成私钥
openssl genpkey -algorithm RSA -out server.key

# 2. 生成证书签名请求（这里会需要输入一些信息：如国家、组织、通用名称等）
openssl req -new -key server.key -out server.csr

# 3. 生成自签名证书
openssl x509 -req -in server.csr -signkey server.key -out server.crt -days 365

```

当然，正常情况下，没必要去单独执行申请 CSR，可以一步到位
```bash
openssl req -x509 -newkey rsa:2048 -keyout server.key -out server.crt -days 365 -nodes

```
* -x509：生成自签名证书。
* -newkey rsa:2048：生成 2048 位的 RSA 私钥。
* -keyout server.key：指定私钥输出文件。
* -out server.crt：指定证书输出文件。
* -days 365：设置证书有效期为 365 天。
* -nodes：不加密私钥。



#### 生成由 CA 签名的证书
如果你有一个 CA (证书颁发机构)，那么可以用 CA 的私钥和证书来签名 CSR

***步骤***
```bash
# 1. 生成私钥
openssl genpkey -algorithm RSA -out server.key

# 2. 生成证书签名请求（这里会需要输入一些信息：如国家、组织、通用名称等）
openssl req -new -key server.key -out server.csr

# 3. 使用CA 签名CSR
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -days 365
```


#### 查看证书内容
生成证书后，可以使用命令查看证书内容
```bash
openssl x509 -in server.crt -text -noout
```

查看私钥内容
```bash
openssl rs -in server.key -text -noout
```

#### 查看证书和私钥是否匹配
```bash
openssl x509 -noout -modules -in server.crt | openssl md5
openssl rsa -noout -modules -in server.key | openssl md5
```
 如果两命令的哈希值一样，则证书和私钥匹配
