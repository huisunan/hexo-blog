---
title: Seata Xa事务超时问题排查和解决
date: 2023-02-25 18:58
tags: java
categories: 
---

<!--more-->

现有服务列表  
![](https://raw.githubusercontent.com/huisunan/cdn/main/img/1410909-20230225185321386-1855228608_1730686602019.png)

在使用seata时总会超时

通过skyWalking追踪发现  
![](https://raw.githubusercontent.com/huisunan/cdn/main/img/1410909-20230225185427826-1128974312_1730686613411.png)

![](https://raw.githubusercontent.com/huisunan/cdn/main/img/1410909-20230225185454578-207629072_1730686613411.png)

在调用接口camunda/group/list时获取getConnection超时，  
在全局事务回滚后，sql又能正常执行

问题原因和解决：

因为没有配置数据库连接池数量，全局事务结束前会挂起connection，服务被频繁调用，把连接池占满，后续新来的请求，没有链接可以使用，造成超时

解决方案：加大连接池数量