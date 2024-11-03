---
title: Sql += 时事务问题
date: 2021-08-25 18:40
tags: 
categories: 
---

<!--more-->

## 表结构

![](https://img2020.cnblogs.com/blog/1410909/202108/1410909-20210825183204141-533674468.png)

## 开启两个会话

会话A和会话B

### 会话A开启事务

![](https://img2020.cnblogs.com/blog/1410909/202108/1410909-20210825183349251-339977943.png)

### 会话B开启事务

![](https://img2020.cnblogs.com/blog/1410909/202108/1410909-20210825183424015-928377680.png)

### 会话A修改值

```sql
update test set value = value + 1 where id = 1;
```

![](https://img2020.cnblogs.com/blog/1410909/202108/1410909-20210825183520571-901495578.png)

### 会话A查询值

```sql
select value from test where id = 1;
```sql
![](https://img2020.cnblogs.com/blog/1410909/202108/1410909-20210825183558512-1669303133.png)


### 会话B查询值
```sql
select value from test where id = 1;
```sql
![](https://img2020.cnblogs.com/blog/1410909/202108/1410909-20210825183638953-1458746126.png)

### 会话B修改值
```sql
update test set value = value + 1 where id = 1;
```

![](https://img2020.cnblogs.com/blog/1410909/202108/1410909-20210825183708298-1495738261.png)

**会话B被阻塞**

### 会话A提交事务

![](https://img2020.cnblogs.com/blog/1410909/202108/1410909-20210825183837154-1483128401.png)

会话B修改成功  
![](https://img2020.cnblogs.com/blog/1410909/202108/1410909-20210825183914350-1998235418.png)

### 会话B提交事务

![](https://img2020.cnblogs.com/blog/1410909/202108/1410909-20210825183939532-798390138.png)