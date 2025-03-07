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
