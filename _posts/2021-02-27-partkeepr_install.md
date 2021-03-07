---
layout: post
title:  "ParKeepr（库存管理软件）安装"
date:   2021-02-27 21:43:00 +0800
categories: [Toolbox, Parkeepr]
tags: [Inventory Management]
---

## 1. ParKeepr 介绍

[Parkeepr](https://partkeepr.org/) 是一款开源的库存管理软件（主要是为电子元器件管理设计的），用来管理手中的电子元器件、开发板是非常不错的。

## 2. 安装

### (1) 安装环境

Ubuntu 20.04.1 LTS, WSL

### (2) 下载安装包

下载安装包：[partkeepr-1.4.0.tbz2](https://downloads.partkeepr.org/partkeepr-1.4.0.tbz2)

```bash
sudo apt-get install php
tar -xvjf partkeepr-1.4.0.tbz2
sudo cp partkeepr-1.4.0 /var/www/partkeepr -r
sudo chown www-data:www-data /var/www/partkeepr -R
# 将当前用户添加到 www-data 组（可能需要重启）：
sudo adduser [username] www-data
```

### (3) 安装依赖软件

安装依赖软件：

```bash
# 可以查看安装包内 documentation/Installation.md 中的说明
sudo apt install mysql-server nginx php-fpm php-curl php-ldap php-bcmath php-gd php-dom php-intl
```

### (4) 配置依赖软件

启动 php-fpm：

```bash
sudo service php-fpm start
```

php-fpm 启动后，可以看到 php-fpm 套接字：

```bash
ls /var/run/php/php*-fpm.sock
```

根据[这个链接](https://wiki.partkeepr.org/wiki/KB00005:Web_Server_Configuration)配置 nginx 服务器。

根据实际情况修改 nginx 配置中的套接字名称，例如：

```
fastcgi_pass unix://var/run/php/php7.4-fpm.sock;
```

设置 php 时区（（需要参照 [https://www.php.net/manual/zh/timezones.php](https://www.php.net/manual/zh/timezones.php)）：

```
# 修改 /etc/php/7.4/fpm/php.ini 文件
date.timezone = Asia/Shanghai
```

之后启动 nginx 以及 mysql，重启 php-fpm

```bash
sudo service nginx start
sudo service mysql start
sudo service php-fpm restart
```

数据库配置：

```bash
sudo mysql -u root mysql

# 创建数据库用户
create user 'partkeepr'@'localhost' identified by 'partkeepr';

# 创建数据库
CREATE DATABASE partkeepr CHARACTER SET UTF8;

# 将数据库授权给创建好的用户
GRANT ALL PRIVILEGES ON partkeepr.* TO partkeepr@localhost;
GRANT USAGE ON *.* TO `partkeepr`@`localhost` IDENTIFIED BY 'partkeepr';
FLUSH PRIVILEGES;
```

将相关服务设置为开机启动：

```bash
sudo systemctl enable nginx
sudo systemctl enable mysql
sudo systemctl enable php7.0-fpm
```

查看设置结果：

```bash
systemctl is-enabled nginx
systemctl is-enabled mysql
systemctl is-enabled php7.0-fpm
```

### (5) 安装 PartKeepr

在浏览器访问安装页面，例如：[http://127.0.0.1/setup/index.html](http://127.0.0.1/setup/index.html)，按照页面呈现的安装步骤操作。如果遇到问题，参见“故障解决”。

安装完成后，根据提示添加定期执行任务：

```bash
crontab -e
```

添加如下内容：

```
0 0,6,12,18 * * * /usr/bin/php /var/www/partkeepr/app/console partkeepr:cron:run
```

上述操作是为当前用户创建定时任务，为了使上述任务顺利运行，需要确保当前用户在 www-data 组中。

## 3. 故障解决

安装 ParKeepr 的过程中，可以查看 nginx 错误日志，有助于发现问题：

```bash
tail -f /var/log/nginx/error.log
```

### (1) Web 服务器配置错误

> Web Server misconfiguration

很可能时时区配置错了（需要参照 [https://www.php.net/manual/zh/timezones.php](https://www.php.net/manual/zh/timezones.php)），如果时区写错了，可以在 nginx 的错误日志中看到。

### (2) 无效的认证码

> Invalid Authentication Key

安装 php7.0（php7.4 版本下，安装 partkeepr 过程中，安装认证无法通过，换成 php7.0 就可以）：

```bash
sudo apt install software-properties-common
sudo apt update
sudo LC_ALL=C.UTF-8 add-apt-repository ppa:ondrej/php 
sudo apt update
sudo apt install php7.0 php7.0-fpm php7.0-curl php7.0-ldap php7.0-bcmath php7.0-gd php7.0-dom php7.0-intl
```

修改 fpm 相关配置以及启动 fpm 时，也需要在相关命令中带上版本号，例如：

```bash
sudo vim /etc/php/7.0/fpm/php.ini
sudo service php7.0-fpm start
```

修改 nginx 配置中的套接字名称

```
fastcgi_pass unix://var/run/php/php7.0-fpm.sock;
```

### (3) 客户端无法识别服务器要求的认证方法

> SQLSTATE[HY000] [2054] The server requested authentication method unknown to the client

这个错可能是 mysql8.0 默认使用 `caching_sha2_password` 作为默认的身份验证插件，而不再是 `mysql_native_password`，但是客户端暂时不支持这个插件导致的。

解决方法：

在 /etc/mysql/my.cnf 文件的结尾新增

```
[mysqld] 
default_authentication_plugin= mysql_native_password
```

重启 mysql：

```bash
sudo service mysql restart
```

修改密码认证方式：

```bash
sudo mysql -u root mysql
# 假定用户名为 partkeepr，密码为 partkeepr
ALTER USER 'partkeepr'@'localhost' IDENTIFIED WITH MYSQL_NATIVE_PASSWORD BY 'partkeepr';
```

### (4) 没有数据库驱动

启用 pdo_mysql 扩展：

```
# 修改 /etc/php/7.4/fpm/php.ini 文件
extension=pdo_mysql
```

重启 mysql 服务，重启后可以用如下命令看到 pdo_mysql

```bash
php7.0 -m
```

## 参考

[1] [https://www.sunzhongwei.com/ubuntu-2004-apt-to-install-php-70](https://www.sunzhongwei.com/ubuntu-2004-apt-to-install-php-70)

[2] [https://askubuntu.com/questions/999999/php-with-pdo-mysql-in-ubuntu-16-04](https://askubuntu.com/questions/999999/php-with-pdo-mysql-in-ubuntu-16-04)

[3] [https://www.jb51.net/article/202399.htm](https://www.jb51.net/article/202399.htm)

[4] [https://github.com/partkeepr/PartKeepr/issues/905](https://github.com/partkeepr/PartKeepr/issues/905)

[5] [https://wiki.partkeepr.org/wiki/PartKeepr_1.4.0_installation_on_a_Raspberry_Pi](https://wiki.partkeepr.org/wiki/PartKeepr_1.4.0_installation_on_a_Raspberry_Pi)

