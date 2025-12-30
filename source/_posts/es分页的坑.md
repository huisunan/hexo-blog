---
title: es分页的坑
date: 2021-06-08 11:21
tags: es
categories: 
---

<!--more-->

这里的form不是第几页的意思，而是偏移量，会导致查询的数据重复

```java
//分页查询
//form不是页码，是偏移量
searchSourceBuilder.from(request.getCurrent().intValue() * request.getSize().intValue());
searchSourceBuilder.size(request.getSize().intValue());

```