---
title: k8s批量调整副本数量
date: 2024-08-22 13:33
tags: 
  - k8s
  - ops
categories: 
---

<!--more-->

deploy

```
kubectl get deploy -n default|tail -q -n 100|awk '{print $1}'|xargs -I ARG kubectl scale deploy ARG -n default --replicas=0

```

statefulset

```
kubectl get statefulset -n default|tail -q -n 100|awk '{print $1}'|xargs -I ARG kubectl scale statefulset ARG -n default --replicas=0

```