---
title: jenkins  mavend mvn too many open files
date: 2023-03-01 22:38
tags: ops
categories: 
---

<!--more-->

先配置mvnd的配置文件 conf/mvnd.properties

```
mvnd.jvmArgs=-XX:-MaxFDLimit
```

jenkins 启动命令添加-XX:-MaxFDLimit

修改系统ulimt的open files 大小