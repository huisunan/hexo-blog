---
title: SpringBoot中JavaMailSender发送附件以及遇到的问题
date: 2021-05-07 13:57
tags: java
categories: 
---

<!--more-->

中文附件名过长变成.bat文件  
解决:  
//防止中文名字 base64加密以后 名字太长被截断 导致中文乱码问题  
System.getProperties\(\).setProperty\("mail.mime.splitlongparameters", "false"\);