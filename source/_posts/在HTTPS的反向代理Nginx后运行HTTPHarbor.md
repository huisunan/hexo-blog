---
title: 在 HTTPS 的反向代理 Nginx 后运行 HTTP Harbor
date: '2025-12-31 10:37:11'
updated: '2025-12-31 10:44:13'
permalink: /post/running-http-harbor-behind-nginx-reverse-proxy-for-https-17qnf6.html
comments: true
toc: true
---



# 在 HTTPS 的反向代理 Nginx 后运行 HTTP Harbor

- **原作者：**   starudream
- **原文来自：**    https://blog.starudream.cn/2022/05/18/harbor-behind-nginx-reverse-proxy/

最终看起来像这样 `nginx (host,ssl)`​ -\> `harbor-nginx (non-ssl)`​ -\> `harbor`。

## 说明

首先服务上安装有 `nginx`​，且配置了 `SSL`​，现在可能在本机或者内网的其他机器上安装有 `Harbor`，需要反向代理到本机映射出去。

### `harbor.yml`

首先需要注释掉 `https`​ 相关的配置，并添加 `external_url` 的配置项。

YAML

```yaml
# Configuration file of Harbor

# The IP address or hostname to access admin UI and registry service.
# DO NOT use localhost or 127.0.0.1, because Harbor needs to be accessed by external clients.
hostname: hub.example.cn

# http related config
http:
  # port for http, default is 80. If https enabled, this port will redirect to https port
  port: 5080

# https related config
# https:
  # https port for harbor, default is 443
  # port: 443
  # The path of cert and key files for nginx
  # certificate: /your/certificate/path
  # private_key: /your/private/key/path

# # Uncomment following will enable tls communication between all harbor components
# internal_tls:
#   # set enabled to true means internal tls is enabled
#   enabled: true
#   # put your cert and key files on dir
#   dir: /etc/harbor/tls/internal

# Uncomment external_url if you want to enable external proxy
# And when it enabled the hostname will no longer used
external_url: https://hub.example.cn
```

### `harbor.conf`

在 `nginx`​ 的 `vhost`​ 中新增相关配置，必须要配置 `X-Forwarded-Proto $scheme`​，`client_max_body_size` 按需配置。

NGINX

```nginx
server {
    listen       80;
    server_name  hub.example.cn;
    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen       443 ssl http2;
    server_name  hub.example.cn;

    ssl_certificate      /usr/local/openresty/nginx/conf/ssl/hub.example.cn.crt;
    ssl_certificate_key  /usr/local/openresty/nginx/conf/ssl/hub.example.cn.key;

    client_max_body_size 500m;

    location / {
        proxy_pass http://10.0.4.10:5080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Real-PORT $remote_port;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

## 常见问题

### `docker login`​ 出现 `unauthorized: authentication required`

​`harbor`​ 内没有配置 `external_url`。

### `docker login`​ 出现 `域名携带的端口丢失`

反向代理 `nginx`​ 中没有配置 `proxy_set_header Host $http_host;`。

### 访问 `hub.example.cn` 会重定向到某个端口

​`harbor`​ 内需要取消 `https` 的配置。

### `docker push`​ 出现 `400 The plain HTTP request was sent to HTTPS port`

反向代理 `nginx`​ 中没有配置 `X-Forwarded-Proto $scheme`。

‍
