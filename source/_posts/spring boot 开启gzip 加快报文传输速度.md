---
title: spring boot 开启gzip 加快报文传输速度
date: 2023-12-30 09:47
tags: java
categories: 
---

<!--more-->

对于一些带宽比较小的服务器，报文内容比较多的应用，这个操作带来的响应是巨大的

```yaml
#配置gzip
server:
  compression:
    enabled: true
    mime-types: application/json
 
```