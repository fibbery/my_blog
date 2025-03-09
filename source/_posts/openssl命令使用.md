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
openssl s_server -cert server.crt -key server.key
```
* ***作用*** ：启动一个 SSL/TLS 服务器，使用指定的证书和私钥。
* ***输出*** ：服务器监听默认端口 4433，等待客户端连接。