---
title: 在 mac 上如何通过 wireshark 调试 https 流量
date: 2025-03-19T23:47:12+08:00
index_img: /images/wireshark/index.png
banner_img: /images/banner.webp
categories: 笔记
excerpt: 使用 wireshark 解密 https 流量
tags:
  - tech
  - tools
---

1. 配置一个本地环境变量 export SSLKEYLOGFILE=$HOME/ssl_key.log
2. 然后在Wireshark的 TLS 中配置主密钥地址，也就是刚才配置的文件地址
{% asset_img ws_master_key_settings.png ws_master_key_settings %}
3. 用命令打开 chrome 或者使用命令执行 curl
```bash
open /Applications/Google\ Chrome.app

# or curl https://www.baidu.com
```
4. 观察$HOME/ssl_key.log 是否新增了值，类似有值就表示成功了
{% asset_img ssl_key_log.png ssl_key_log%}
> 日志有多种格式，tls1.2的格式是 CLIENT_RANDOM {CLIENT_RANDOM} {MASTER_SECRECT}
> 通过 CLIENT_RANDOM 定位会话，使用 MASTER_SECRECT 解密会话数据

## 原理解析
SSLKEYLOGFILE 的原理是利用浏览器（如 Chrome、Firefox）或应用程序提供的调试功能，将密钥导出到日志文件，从而可以用这些密钥解密 TLS 流量。
> 参考 ssl 握手图，实际就是在生成 MASTER_SECRECT 之后写入日志文件
{% asset_img ssl_handshake.png ssl_handshake%}



## Reference
[How to Decrypt SSL with Wireshark – HTTPS Decryption Guide](https://www.comparitech.com/net-admin/decrypt-ssl-with-wireshark/)