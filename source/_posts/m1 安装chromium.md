---
title: m1 安装chromium
date: 2022-01-22 10:35
tags: mac
categories: 
---

<!--more-->

```shell
brew install --cask chromium
```

如果下载失败，可以使用代理下载

下载后chromium打开失败，可以消除app的隔离属性

```shell
sudo xattr -r -d com.apple.quarantine /Applications/Chromium.app
```