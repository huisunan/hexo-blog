---
title: jenkins + sonar 中文文件名报错解决
date: 2021-05-10 17:33
tags: ops
categories: 
---

<!--more-->

[![gNekz8.png](https://z3.ax1x.com/2021/05/10/gNekz8.png)](https://imgtu.com/i/gNekz8)  
在Jvm options 中加入参数

```
-Dsun.jnu.encoding=UTF-8 -Dfile.encoding=UTF-8
```