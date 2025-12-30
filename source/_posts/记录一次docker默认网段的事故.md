---
title: 记录一次docker默认网段的事故
date: 2023-01-04 19:30
tags: ops
categories: 
---

<!--more-->

docker的默认网段使用的是172.17.0.0/24和办公网起了冲突，导致办公网无法访问

修改默认网段

修改/etc/docker/daemon.json

```json
{
  "bip": "192.168.210.1/24"
}
```