---
title: 使用RSS+n8n同步博客园文章到cubox
date: 2024-02-07 22:44
tags: other
categories: 
---

<!--more-->

# 使用RSS+n8n同步博客文章到Cubox

## Cubox

Cubox是一款碎片知识文章收集的应用

![](https://raw.githubusercontent.com/huisunan/cdn/main/img/1410909-20240207223738974-1361379525_1730686602019.png)

## n8n

低代码的workFlow

![](https://raw.githubusercontent.com/huisunan/cdn/main/img/1410909-20240207223759173-1889192280_1730686602019.png)

## 整合

![](https://raw.githubusercontent.com/huisunan/cdn/main/img/1410909-20240207224237831-1340391436_1730686602019.png)

大致流程

定时触发器->获取RSS列表->迭代->文章是否已经同步->同步文章到cubox->同步记录写到数据库->结束

> 这是一个大概的流程，当然也可以实现同步到其他地方的流程