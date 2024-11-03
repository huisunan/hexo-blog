---
title: Mysql not in 不走索引
date: 2022-02-15 19:26
tags: 
categories: 
---

<!--more-->

## 测试数据和索引

![](https://img2022.cnblogs.com/blog/1410909/202202/1410909-20220215191248745-1664519830.png)  
![](https://img2022.cnblogs.com/blog/1410909/202202/1410909-20220215191305187-1158856986.png)

### MySQL5.7

打印执行计划，type是all走的全表  
![](https://img2022.cnblogs.com/blog/1410909/202202/1410909-20220215191404640-1699744545.png)

### MySQL8.0

type是range对索引进行范围扫描  
![](https://img2022.cnblogs.com/blog/1410909/202202/1410909-20220215192226393-1386245983.png)

### MySQL5.7解决方案

使用覆盖索引代替，not in就可以走索引了  
![](https://img2022.cnblogs.com/blog/1410909/202202/1410909-20220215192456122-1965915254.png)  
![](https://img2022.cnblogs.com/blog/1410909/202202/1410909-20220215192605877-1433106593.png)