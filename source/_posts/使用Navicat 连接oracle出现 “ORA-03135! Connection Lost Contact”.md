---
title: 使用Navicat 连接oracle出现 “ORA-03135 Connection Lost Contact”
date: 2021-05-07 11:06
tags: other
categories: 
---

<!--more-->

我使用的是navicat premium版本，之所以用这个是为了能导出数据库表，在连接数据库时候，出现了 “ORA-03135: Connection Lost Contact”，这个是因为navicat通常会在自己的安装路径下包含某个版本的OCI，如果使用navicat连接Oracle服务器时出现ORA-03135错误，大部分是因为navicat本地的OCI版本与Oracle服务器器不符造成的。解决方法就是去OCI的下载页面

<http://www.oracle.com/technetwork/database/features/instant-client/index-097480.html>

下载指定版本的oci，我直接下载的是

这个版本，下载后直接点击Navicat —工具–选项–其他–OCI

配置OCI library为刚下载的zip包解压后的跟目录，有个oci文件，直接选中它 ，点击确定–重启Navicat，再次打开Navicat，便显示连接成功