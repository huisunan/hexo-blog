---
title: 多线程之BlockingQueue中 take、offer、put、add的一些比较
date: 2021-05-07 11:32
tags: java
categories: 
---

<!--more-->

BlockingQueue 方法以四种形式出现，对于不能立即满足但可能在将来某一时刻可以满足的操作，这四种形式的处理方式不同：第一种是抛出一个异常，第二种是返回一个特殊值（null 或 false，具体取决于操作），第三种是在操作可以成功前，无限期地阻塞当前线程，第四种是在放弃前只在给定的最大时间限制内阻塞。第四种的返回值和第二种的一样。  
下表中总结了这些方法：

|  | 抛出异常 | 特殊值 | 阻塞 | 超时 |
| --- | --- | --- | --- | --- |
| 插入 | add\(e\) | offer\(e\) | put\(e\) | offer\(e, time, unit\) |
| 移除 | remove\(\) | poll\(\) | take\(\) | poll\(time, unit\) |
| 检查 | element\(\) | peek\(\) | 不可用 | 不可用 |