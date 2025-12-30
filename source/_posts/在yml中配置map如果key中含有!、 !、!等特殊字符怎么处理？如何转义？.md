---
title: 在yml中配置map如果key中含有/、 \、*等特殊字符怎么处理？如何转义？
date: 2021-05-07 10:06
tags: java
categories: 
---

<!--more-->

在yml配置map如果key中含有 \\ \* 等特殊字符，key 需要加 "\[ \]"  
[![g1DaFS.png](https://z3.ax1x.com/2021/05/07/g1DaFS.png)](https://imgtu.com/i/g1DaFS)

```yaml
filter:
  filterChainDefinitionMap:
   {"[/advertising/*]": 'perms[公告管理]',
    "[/hotelmanagement/*]": 'perms[入住管理]',
    "[/broadband/*]": 'perms[报装报修]',
    "[/yellowpages/*]": 'perms[黄页管理]'}
```