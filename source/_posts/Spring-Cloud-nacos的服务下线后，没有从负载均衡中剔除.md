---
title: Spring Cloud nacos的服务下线后，没有从负载均衡中剔除
date: 2025-04-28 18:56:01
tags: 
  - java
---

通过debug发现，Spring Cloud的负载均衡对服务进行了缓存，导致nacos的服务从nacos中下线，没有及时剔除

org.springframework.cloud.loadbalancer.core.CachingServiceInstanceListSupplier

解决方法

关闭该缓存，nacos自身是有缓存的
```yaml
spring:
  cloud:
    # 禁用loadbalancer缓存,实现优雅下线
    loadbalancer:
      cache:
        enabled: false
```