---
title: 绕过new bing限制
date: 2023-04-29 12:42:31
tags:
---

下载Edge

安装header editor插件

创建规则
* Rule type: Modify Request Header
* Match type: Regular Expression
* Match Rules: ^http(s?)://www.bing.com/(.*)
* Header Name: x-forwarded-for
* Header Value: 1.1.1.1

感谢[@starzqeth](https://twitter.com/starzqeth)