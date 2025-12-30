---
title: Mybatis传入list的注意点
date: 2021-05-07 11:40
tags: java
categories: 
---

<!--more-->

**当mybatis传入参数为list集合的时候；mybatis会自动把其封装为一个map；会以“list”作为key**

```language
    每个元素的值作为value；格式为Map<"list",value>
    当mybatis传入参数为数组的时候mybatis会自动把其封装为一个map；会以“array”作为key；
    每个元素的值作为value；格式为Map<"array",value>
```