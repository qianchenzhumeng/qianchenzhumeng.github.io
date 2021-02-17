---
layout: post
title:  "Nginx 反向代理 Thingsboard 并缓存静态文件"
date:   2018-02-08 20:45:00 +0800
categories: [IoT, Server]
tags: [ThingsBoard, Nginx]
---

## 1. 背景

使用浏览器访问 ThingsBoard 时速度较慢，原因是有个 CSS 文件体积较大（超过 2 M）。

## 2. 安装配置 Nginx

Centos 安装 Nginx：

```
# yum install nginx
```

创建缓存目录：

```
mkdir -pv /cache/nginx/
chown -R nginx:nginx /cache/nginx
```

创建配置文件：

```
vim /etc/nginx/conf.d/thingsboard.conf
```
```
proxy_cache_path /cache/nginx/ levels=1:2 keys_zone=STATIC:50m max_size=2G inactive=365d;
server {
    listen 80;
    listen [::]:80;

    server_name 59.110.231.79;
    
    location ~*  \.(jpg|jpeg|png|gif|ico|css|js|woff2)$ {
        proxy_cache STATIC;
        proxy_pass http://localhost:8080;
        proxy_set_header Host $host;
        proxy_cache_valid 200 1d;
        proxy_cache_use_stale  error timeout invalid_header updating
                                   http_500 http_502 http_503 http_504;
        proxy_ignore_headers X-Accel-Expires Expires Cache-Control Set-Cookie;
        proxy_hide_header Pragma;
        expires 365d;
    }

    location / {
        proxy_pass http://localhost:8080/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
    }
}
```

重启nginx服务：

```
# service nginx restart
```

## 参考

[1] [https://www.nginx.com/blog/nginx-caching-guide/](https://www.nginx.com/blog/nginx-caching-guide/)

[2] [https://linode.com/docs/development/iot/install-thingsboard-iot-dashboard/](https://linode.com/docs/development/iot/install-thingsboard-iot-dashboard/)

[3] [https://www.howtoforge.com/make-browsers-cache-static-files-on-nginx](https://www.howtoforge.com/make-browsers-cache-static-files-on-nginx)
