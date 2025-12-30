---
title: k8s-nfs
date: 2022-12-05 13:36
tags:
  - k8s
  - ops
categories: 
---

<!--more-->

nfs.server nfs服务器

```

helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/

helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
    --set nfs.server=192.168.1.121 \
    --set nfs.path=/data \
    --set image.repository=dyrnq/nfs-subdir-external-provisioner

```