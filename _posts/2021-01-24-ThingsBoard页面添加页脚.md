---
layout: post
title:  "ThingsBoard页面添加页脚"
date:   2021-01-24 08:23:00 +0800
categories: [IoT]
tags: [ThingsBoard]
---

## 1. 背景

需要在 ThingsBoard 页面下面添加页脚标签，首先想到的是可不可以修改配置文件或者安装目录内的一些 html 文件来完成添加，检索之后发现该方法不可行，原因是 html 页面都被编译到 `thingsboard.jar` 文件中去了，无法直接修改。另一种思路就是修改 ThingsBoard 的源码，然后再编译。

## 2. 添加页脚标签

下载源码：https://github.com/thingsboard/thingsboard

检出所需版本，例如：

```shell
git checkout release-3.2
```

修改 `ui-ngx/src/index.html` 文件，添加 `footer` 标签，例如（原有的某些部分在这里使用 ... 代替显示）：

```html
<!doctype html>
<html lang="en" style="width: 100%;">
<head>
  ...
</head>
<body class="tb-default">
  ...
  <div class="footer">
    <li>
      <a>&copy; 2021 XXXX</a>
    </li>
  </footer>
</body>
</html>

<style >  
  .footer{
    position:absolute;
    left: 0;
    right: 0;
    bottom:0;   /* 位于底部 */
    text-align:center;  /* 居中显示 */
    z-index: 30;    /* 位于上层 */
  }
</style>
```

其他的页面都会使用这个页面作为模板，所以只需要修改这个文件即可。

## 3. 从源码编译 ThingsBoard

按官方指导文档编译 ThingsBoard：https://thingsboard.io/docs/user-guide/install/building-from-source/

注意，如果本地已经安装并启动了 ThingsBoard 以及 Postgres，需要将二者关闭，否则编译源码过程中进行的单元测试会受影响（端口被占用）。

如果是 windows，停止相关的服务，例如：

- ThingsBoard Server Application
- postgresql-x64-11 - PostgreSQL Server 11



