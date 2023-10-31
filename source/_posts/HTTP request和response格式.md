---
title: HTTP request和response格式
date: 2023-10-31T15:18:39+08:00
categories: 笔记
excerpt: 介绍Http request和response格式区别
tags:
  - tech
---

# HTTP 请求报文
一个 HTTP 请求报文由以下几部分组成
1. request line 请求行
2. request header 请求头部
3. blank line 空行
4. request body 请求体
{% asset_img request.png request %}

{% note primary %}
HTTP请求方法可选: GET、POST、HEAD、PUT、DELETE、OPTIONS、TRACE、CONNECT  
HTTP协议版本可选: HTTP/1.1
{% endnote %}

# HTTP 响应报文
一个 HTTP 响应报文由以下几部分组成
1. status line 状态行
2. response header 响应头部
3. blank line 空行
4. reponse body 响应体
{% asset_img response.png response %}

{% note primary %}
其中状态码由三位数字组成，状态码描叙用来描叙状态码的信息。状态码的第一位数字定义了响应类别，只有 5 种取值：
- 1xx：指示信息--表示请求已接收，继续处理。
- 2xx：成功--表示请求已被成功接收、理解、接受。
- 3xx：重定向--要完成请求必须进行更进一步的操作。
- 4xx：客户端错误--请求有语法错误或请求无法实现。
- 5xx：服务器端错误--服务器未能实现合法的请求。
{% endnote %}
{% note primary %} 
常见状态码以及说明如下：
- 200 OK：客户端请求成功。
- 400 Bad Request：客户端请求有语法错误，不能被服务器所理解。
- 401 Unauthorized：请求未经授权，这个状态代码必须和 WWW-Authenticate 报头域一起使用。
- 403 Forbidden：服务器收到请求，但是拒绝提供服务。
- 404 Not Found：请求资源不存在，举个例子：输入了错误的 URL。
- 500 Internal Server Error：服务器发生不可预期的错误。
- 503 Server Unavailable：服务器当前不能处理客户端的请求，一段时间后可能恢复正常，举个例子：HTTP/1.1 200 OK（CRLF）。
{% endnote %}