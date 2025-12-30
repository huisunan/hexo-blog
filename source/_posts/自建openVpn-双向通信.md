---
title: 自建openVpn 双向通信
date: 2025-06-28 22:58:40
tags:
  - openvpn
---

已经搭建好openvpn as服务器,此文章主要解决双向通信的问题

openvpn as 如果是docker部署的，推荐network_mode修改为host


> 公司网段: 10.88.0.0/19
>
>vpn网段: 10.88.32.0/20
> 
> openvpn ip: 10.88.1.4



目前 vpn网段已经支持

## 添加vpn客户端访问公司网段的权限

![](../images/vpn-setting.png)

## 添加公司网段内机器访问openvpn客户端机器

在路由器上添加路由表

10.88.32.0/20 --->  10.88.1.4

openvpn所在机器开启 

服务器上开启转发

```shell
echo 'net.ipv4.ip_forward = 1' >> /etc/sysctl.conf
sysctl -p
```

查看一下路由表

![](../images/vpn-setting-2.png)


在AS0_OUT_S2C 前添加一个转发策略

```shell
iptables -I FORWARD 3 -d 10.88.32.0/20 -j ACCEPT
```

## 为什么在AS0_OUT_S2C前添加转发策略

![img.png](../images/vpn-setting-3.png)

追踪一下链路会发现，请求中没有标识的会被直接drop掉，导致转发失败