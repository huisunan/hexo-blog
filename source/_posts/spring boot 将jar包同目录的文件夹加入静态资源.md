---
title: spring boot 将jar包同目录的文件夹加入静态资源
date: 2021-10-25 14:15
tags: 
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

![](https://img2020.cnblogs.com/blog/1410909/202110/1410909-20211025141404797-129654207.png)

效果：  
![](https://img2020.cnblogs.com/blog/1410909/202110/1410909-20211025141429683-1984817132.png)