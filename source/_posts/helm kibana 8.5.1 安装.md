---
title: helm kibana 8.5.1 安装
date: 2022-12-06 01:09
tags: es
categories: 
---

<!--more-->

helm应用名使用kibana

清除命令

```shell
kubectl delete configmap kibana-kibana-helm-scripts -n elastic
kubectl delete serviceaccount pre-install-kibana-kibana -n elastic
kubectl delete roles pre-install-kibana-kibana -n elastic
kubectl delete rolebindings pre-install-kibana-kibana -n elastic
kubectl delete job pre-install-kibana-kibana -n elastic

kubectl delete serviceaccount post-delete-kibana-kibana -n elastic
kubectl delete roles post-delete-kibana-kibana -n elastic
kubectl delete rolebindings post-delete-kibana-kibana -n elastic
kubectl delete job post-delete-kibana-kibana -n elastic
```