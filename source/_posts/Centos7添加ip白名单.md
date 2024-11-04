---
title: Centos7添加ip白名单
date: 2022-06-16 19:58
tags: linux
categories: 
---

<!--more-->

```shell
firewall-cmd --permanent --add-rich-rule='rule family=ipv4 source address=10.255.19.3 accept'
```