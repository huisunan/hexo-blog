---
title: Mysql Json操作
date: 2021-08-28 09:55
tags: java
categories: 
---

<!--more-->

# Json数组

## 数组包含某个数据

![](https://raw.githubusercontent.com/huisunan/cdn/main/img/1410909-20210828094049059-46282987_1730686650382.png)

```sql
SELECT * from test WHERE JSON_CONTAINS(`value`, '3','$');
```

![](https://raw.githubusercontent.com/huisunan/cdn/main/img/1410909-20210828094121830-1738574518_1730686650382.png)

## 获取长度

```sql
SELECT JSON_LENGTH(`value`,'$') from test
```

![](https://raw.githubusercontent.com/huisunan/cdn/main/img/1410909-20210828095430267-1737547699_1730686650382.png)