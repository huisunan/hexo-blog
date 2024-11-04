---
title: jdk LocalDateTime mybatis 空指针解决办法
date: 2021-05-07 11:27
tags: java
categories: 
---

<!--more-->

引入jar包：

1.  mysql.mysql-connector-java:5.1.39

2.  org.mybatis.mybatis:3.5.2

3. org.mybatis.mybatis-spring:2.0.2

在项目中的mybats升级使用了jdk8的LocalDateTime等后，数据库timesstamp字段有的记录是null，导致查询时出现下面错误

```java
org.mybatis.spring.MyBatisSystemException: nested exception is org.apache.ibatis.executor.result.ResultMapException: Error attempting to get column 'UPDATE_TIME' from result set.  Cause: java.lang.NullPointerException
	at org.mybatis.spring.MyBatisExceptionTranslator.translateExceptionIfPossible(MyBatisExceptionTranslator.java:78)
	at org.mybatis.spring.SqlSessionTemplate$SqlSessionInterceptor.invoke(SqlSessionTemplate.java:440)
	at com.sun.proxy.$Proxy154.selectList(Unknown Source)
	at org.mybatis.spring.SqlSessionTemplate.selectList(SqlSessionTemplate.java:223)
	at org.apache.ibatis.binding.MapperMethod.executeForMany(MapperMethod.java:147)
	at org.apache.ibatis.binding.MapperMethod.execute(MapperMethod.java:80)
	at org.apache.ibatis.binding.MapperProxy.invoke(MapperProxy.java:57)
	at com.sun.proxy.$Proxy181.find(Unknown Source)
	at cn.enn.ygego.sunny.sv.service.online.impl.LogisticsBrandServiceImpl.list(LogisticsBrandServiceImpl.java:67)
	at cn.enn.ygego.sunny.sv.controller.online.LogisticsBrandController.list(LogisticsBrandController.java:49)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
```

不能把null转换为LocalDateTime。通过跟踪代码，发现问题出在mysql的驱动上，JDBC42ResultSet的getObject代码如下：

```java
public <T> T getObject(int columnIndex, Class<T> type) throws SQLException {
        if (type == null) {
            throw SQLError.createSQLException("Type parameter can not be null", "S1009", this.getExceptionInterceptor());
        } else if (type.equals(LocalDate.class)) {
            return type.cast(this.getDate(columnIndex).toLocalDate());
        } else if (type.equals(LocalDateTime.class)) {
            return type.cast(this.getTimestamp(columnIndex).toLocalDateTime());
        } else if (type.equals(LocalTime.class)) {
            return type.cast(this.getTime(columnIndex).toLocalTime());
        } else {
            if (type.equals(OffsetDateTime.class)) {
                try {
                    return type.cast(OffsetDateTime.parse(this.getString(columnIndex)));
                } catch (DateTimeParseException var5) {
                }
            } else if (type.equals(OffsetTime.class)) {
                try {
                    return type.cast(OffsetTime.parse(this.getString(columnIndex)));
                } catch (DateTimeParseException var4) {
                }
            }
 
            return super.getObject(columnIndex, type);
        }
    }
```

```java
return type.cast(this.getTimestamp(columnIndex).toLocalDateTime());代码中this.getTimestamp(columnIndex)返回null，再次执行toLocalDateTime()，当然报错。
```

解决方式升级mysql驱动，我升级到5.1.47，其他版本没有测试，在5.1.47中代码如下：

```java
public <T> T getObject(int columnIndex, Class<T> type) throws SQLException {
        if (type == null) {
            throw SQLError.createSQLException("Type parameter can not be null", "S1009", this.getExceptionInterceptor());
        } else if (type.equals(LocalDate.class)) {
            Date date = this.getDate(columnIndex);
            return date == null ? null : type.cast(date.toLocalDate());
        } else if (type.equals(LocalDateTime.class)) {
            Timestamp timestamp = this.getTimestamp(columnIndex);
            return timestamp == null ? null : type.cast(timestamp.toLocalDateTime());
        } else if (type.equals(LocalTime.class)) {
            Time time = this.getTime(columnIndex);
            return time == null ? null : type.cast(time.toLocalTime());
        } else {
            String string;
            if (type.equals(OffsetDateTime.class)) {
                try {
                    string = this.getString(columnIndex);
                    return string == null ? null : type.cast(OffsetDateTime.parse(string));
                } catch (DateTimeParseException var5) {
                }
            } else if (type.equals(OffsetTime.class)) {
                try {
                    string = this.getString(columnIndex);
                    return string == null ? null : type.cast(OffsetTime.parse(string));
                } catch (DateTimeParseException var4) {
                }
            }
 
            return super.getObject(columnIndex, type);
        }
    }
```