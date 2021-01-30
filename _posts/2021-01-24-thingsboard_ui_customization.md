---
layout: post
title:  "ThingsBoard UI 修改"
date:   2021-01-24 08:23:00 +0800
categories: [IoT]
tags: [ThingsBoard]
---

## 1. 背景

需要在 ThingsBoard 页面下面添加页脚标签，首先想到的是可不可以修改配置文件或者安装目录内的一些 html 文件来完成添加，检索之后发现该方法不可行，原因是 html 页面都被编译到 `thingsboard.jar` 文件中去了，无法直接修改。另一种思路就是修改 ThingsBoard 的源码，然后再编译。

版本：3.1

## 2. 添加页脚标签

下载源码：https://github.com/thingsboard/thingsboard

检出所需版本，例如：

```shell
git checkout release-3.1
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

## 3. 替换标题

修改以下文件中的 `appTitle` 的值:

- ui-ngx/src/environments/environment.prod.ts
- ui-ngx/src/environments/environment.ts

修改以下文件中 `title` 标签的内容

- ui-ngx/src/index.html

## 4. 修改页面语言

修改以下文件中的 `defaultLang` 的值:

- ui-ngx/src/environments/environment.prod.ts
- ui-ngx/src/environments/environment.ts

```
defaultLang: 'zh_CN'
```

## 5. 从源码编译 ThingsBoard

如果需要生成安装包，按官方指导文档编译 ThingsBoard[[1]](https://thingsboard.io/docs/user-guide/install/building-from-source/)：

```shell
sudo apt-get install maven
```

编译：

```shell
# clean 会删除 target，没必要的话不用加。
mvn clean install
# 或者
mvn clean package
```

产物位于 `application/target`。

如果已经安装过了，使用 `application/target/thingsboard-3.1.1-boot.jar` 替换 thingsboard.jar 即可：

- windows: thingsboard/lib/thingsboard.jar
- ubuntu: /usr/share/thingsboard/bin/thingsboard.jar

## 6. 故障解决

### (1) yarn 网络连接问题

> [INFO] --- frontend-maven-plugin:1.7.5:yarn (yarn install) @ ui-ngx ---
> [INFO] Running 'yarn install' in /mnt/f/wsl/source/thingsboard/ui-ngx
> [INFO] yarn install v1.22.4
> [INFO] [1/4] Resolving packages...
> [INFO] [2/4] Fetching packages...
> [INFO] info There appears to be trouble with your network connection. Retrying...
> [INFO] info There appears to be trouble with your network connection. Retrying...

设置代理：

```shell
# https
./ui-ngx/target/node/yarn/dist/bin/yarn config set https-proxy http://127.0.0.1:50657
# http
./ui-ngx/target/node/yarn/dist/bin/yarn config set proxy http://127.0.0.1:50657
```

或修改 yarn 源：

```shell
./ui-ngx/target/node/yarn/dist/bin/yarn config get registry
# http://registry.npmjs.org/
./ui-ngx/target/node/yarn/dist/bin/yarn config set registry http://registry.npm.taobao.org/
# yarn config v1.22.4
# success Set "registry" to "http://registry.npm.taobao.org/".
# Done in 0.03s.
# 继续编译
mvn install -rf :ui-ngx
```

### (2) 端口占用

如果本地已经安装并启动了 ThingsBoard 以及 Postgres，需要将二者关闭，否则编译源码过程中进行的单元测试会受影响（端口被占用）。

如果是 windows，停止相关的服务，例如：

- ThingsBoard Server Application
- postgresql-x64-11 - PostgreSQL Server 11

## 参考

[1] https://thingsboard.io/docs/user-guide/install/building-from-source/

[2] https://blog.csdn.net/qgbihc/article/details/108969424

