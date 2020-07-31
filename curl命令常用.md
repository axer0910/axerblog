---
title: Curl命令常用笔记
tag: server
date: 2019-03-06
updated: 2019-03-06
---

curl命令可以很方便的创建一些请求脚本，支持各种请求方式，自定义头部，指定Agent。

`curl -A "[agent]" -H "[header]" -X POST -d "[rawbody]" [url]`

*  **-A "[agent]"：agent可以指定浏览器信息。**
*  **-H "[header]"：header可以指定头部信息（类似Content-Type：application/x-www-form-urlencoded）。多个头部就添加多个 -H "[header]" ...**
*  **-X POST 请求方法：GET, POST, PUT, DELETE ...**
*  **-d "[rawbody]"：http的body，如果是POST或者PUT就放请求的body，注意编码格式（content-type是form就放form格式编码后的结果，json就放json字符串）**
*  [url]：请求的地址。

保存到一个文件，可以使用linux的重定向命令：
```
curl https://baidu.com >> baidu.html
```