---
title: spring boot 将jar包同目录的文件夹加入静态资源
date: 2021-10-25 14:15
tags: java
categories: 
---

<!--more-->

配置

```yaml
spring:
  web:
    resources:
      static-locations: file:upload
```

![](https://raw.githubusercontent.com/huisunan/cdn/main/img/1410909-20211025141404797-129654207_1730686630335.png)

效果：  
![](https://raw.githubusercontent.com/huisunan/cdn/main/img/1410909-20211025141429683-1984817132_1730686630335.png)