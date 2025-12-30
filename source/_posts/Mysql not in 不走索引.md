---
title: Mysql not in 不走索引
date: 2022-02-15 19:26
tags: mysql
categories: 
---

<!--more-->

## 测试数据和索引

![](https://raw.githubusercontent.com/huisunan/cdn/main/img/1410909-20220215191248745-1664519830_1730686613411.png)  
![](https://raw.githubusercontent.com/huisunan/cdn/main/img/1410909-20220215191305187-1158856986_1730686621846.png)

### MySQL5.7

打印执行计划，type是all走的全表  
![](https://raw.githubusercontent.com/huisunan/cdn/main/img/1410909-20220215191404640-1699744545_1730686621846.png)

### MySQL8.0

type是range对索引进行范围扫描  
![](https://raw.githubusercontent.com/huisunan/cdn/main/img/1410909-20220215192226393-1386245983_1730686621847.png)

### MySQL5.7解决方案

使用覆盖索引代替，not in就可以走索引了  
![](https://raw.githubusercontent.com/huisunan/cdn/main/img/1410909-20220215192456122-1965915254_1730686621847.png)  
![](https://raw.githubusercontent.com/huisunan/cdn/main/img/1410909-20220215192605877-1433106593_1730686621847.png)