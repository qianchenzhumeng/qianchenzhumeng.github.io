---
layout: post
title:  "加快访问 Coursera 的速度"
date:   2021-04-18 18:19:00 +0800
categories: [Other]
tags: [Other]
---

大多数情况下，在国内访问 Coursera 会比较慢，甚至会出现视频加载失败的问题。这是因为 DNS 被污染了，可以通过修改 hosts 的方式解决 DNS 污染问题。

在这个 [IPAddress.com](https://www.ipaddress.com/ip-lookup) 网站上查找如下三个域名对应的 IP 地址（一个域名可能会对应多个 IP 地址）：

- www.coursera.org
- d3c33hcgiwev3.cloudfront.net
- d3njjcbhbojbot.cloudfront.net

如果是 Windows，使用记事本打开 C:\Windows\System32\drivers\etc\hosts 文件，将查到的 IP 地址和域名追加到该文件中，例如：

```
# coursera hosts 
13.226.13.73   www.coursera.org
13.226.13.10    www.coursera.org
13.226.13.76    www.coursera.org
13.226.13.59    www.coursera.org

54.192.130.170  d3c33hcgiwev3.cloudfront.net
54.192.130.124  d3c33hcgiwev3.cloudfront.net
54.192.130.80   d3c33hcgiwev3.cloudfront.net
54.192.130.189  d3c33hcgiwev3.cloudfront.net

54.230.18.14    d3njjcbhbojbot.cloudfront.net
54.230.18.111   d3njjcbhbojbot.cloudfront.net
54.230.18.122   d3njjcbhbojbot.cloudfront.net
54.230.18.93    d3njjcbhbojbot.cloudfront.net
```

保存退出即可。