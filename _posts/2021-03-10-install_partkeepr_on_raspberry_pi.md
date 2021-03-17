---
layout: post
title:  "在树莓派上安装 ParKeepr"
date:   2021-03-10 20:05:00 +0800
categories: [Toolbox, Parkeepr]
tags: [Inventory Management]
---

## 1. 背景

本文是对官方 wiki 的粗略翻译（原文见[[1](https://wiki.partkeepr.org/wiki/PartKeepr_1.4.0_installation_on_a_Raspberry_Pi)]），文中使用 Apache 作为服务器。我试过使用 Nginx + php7.3-fpm 的组合，没有成功，安装进行到预热缓存的时后老是失败，估计对树莓派来说负荷有点重。

## 2. 安装

### (1) 安装环境

树莓派 3 B+，Raspbian GNU/Linux 10 

### (2) 下载安装包

下载安装包：[partkeepr-1.4.0.tbz2](https://downloads.partkeepr.org/partkeepr-1.4.0.tbz2)

```bash
sudo apt-get install php
tar -xvjf partkeepr-1.4.0.tbz2
sudo cp partkeepr-1.4.0 /var/www/partkeepr -r
sudo chown www-data:www-data /var/www/partkeepr -R
# 将当前用户添加到 www-data 组（可能需要重启）：
sudo adduser pi www-data
```

### (3) 安装依赖软件

安装 PHP：

```bash
sudo apt-get install php7.3 php-bcmath php7.3-curl php7.3-gd php7.3-intl php7.3-ldap php7.3-mysql php7.3-xml php-pgsql php-apcu php-apcu-bc 
```

安装 mysql：

```bash
sudo apt-get install mariadb-server-10.0 mariadb-client-10.0
```

安装 Apache 服务器：

```bash
sudo apt-get install apache2
```

安装 curl 和 ntp：

```bash
sudo apt-get install curl ntp
```

### (4) 配置依赖软件

**配置 Apache**

使能 Apache 的 userdir 和 rewrite 模块：

```bash
sudo a2enmod userdir rewrite
```

修改 Apache 设置：

```bash
sudo vim /etc/apache2/apache2.conf
```

添加如下内容：

```
<Directory /var/www/partkeepr/>
	AllowOverride All
</Directory>
```

修改 Apache 根目录：

```bash
sudo vim /etc/apache2/sites-available/000-default.conf
```

```
# DocumentRoot /var/www/html
DocumentRoot /var/www/partkeepr/web
```

```bash
sudo vim /etc/apache2/sites-available/default-ssl.conf
```

```
# DocumentRoot /var/www/html
DocumentRoot /var/www/partkeepr/web
```

**配置 PHP**

```bash
sudo vim /etc/php/7.3/apache2/php.ini
```

设置 php 时区（需要参照 [https://www.php.net/manual/zh/timezones.php](https://www.php.net/manual/zh/timezones.php)）：

```ini
;date.timezone =
date.timezone = Asia/Shanghai
```

修改最大执行时间（秒）：

```ini
;max_execution_time = 30
max_execution_time = 150
```

修改文件上传容量（按需修改）：

```ini
;upload_max_filesize = 2M
upload_max_filesize = 1024M
```

**APC 元数据缓存**

```bash
sudo vim /var/www/partkeepr/app/config/config_framework.yml
```

在最上面添加如下内容：

```yml
services:
    app.doctrine.apc_cache:
       class: Doctrine\Common\Cache\ApcCache
       calls:
           - [setNamespace, [""]]
```

在最后添加如下内容：

```yaml
framework:
    annotations:
        cache: "app.doctrine.apc_cache"
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

重启服务器：

```bash
sudo service apache2 start
```

将相关服务设置为开机启动：

```bash
sudo systemctl enable mysql
sudo systemctl enable apache2
```

查看设置结果：

```bash
sudo systemctl is-enabled mysql
sudo systemctl is-enabled apache2
```

### (5) 安装 PartKeepr

在浏览器访问安装页面，假设树莓派 ip 是 192.168.1.2，访问：[http://192.168.1.2/setup/index.html](http://192.168.1.2/setup)，按照页面呈现的安装步骤操作。如果遇到问题，参见“故障解决”。

安装完成后，根据提示添加定期执行任务：

```bash
crontab -e
```

添加如下内容：

```
0 0,6,12,18 * * * /usr/bin/php /var/www/partkeepr/app/console partkeepr:cron:run
```

上述操作是为当前用户创建定时任务，为了使上述任务顺利运行，需要确保当前用户在 www-data 组中。

## 3. 备份数据库

树莓派还是比较脆弱，最好经常备份一下数据库：

```bash
mysqldump -upartkeepr -ppartkeepr partkeepr > partkeepr_2021_03_17_20_31.mysqldump
```

导出后放在其他设备上。

## 4. 故障解决

### (1) 预热缓存时报错

> Invalid configuration for path "monolog.handlers.main": Warning: count(): Parameter must be an array or an object that implements Countable

需要修改文件[[2](https://github.com/partkeepr/PartKeepr/issues/905)]：

```bash
sudo vim /var/www/partkeepr/vendor/symfony/monolog-bundle/DependencyInjection/Configuration.php
```

替换第 594 行的内容：

```php
// ->ifTrue(function ($v) { return ('fingers_crossed' === $v['type'] || 'buffer' === $v['type'] || 'filter' === $v['type']) && 1 !== count($v['handler']); })
->ifTrue(function ($v) { return ('fingers_crossed' === $v['type'] || 'buffer' === $v['type'] || 'filter' === $v['type']) && (empty($v['handler']) || !is_string($v['handler'])); })
```

> Invalid Response from server

遇到这个问题，多试几次……不行的话重启再试几次，不要急，毕竟树莓派性能摆在那里。

## 参考

[1] [https://wiki.partkeepr.org/wiki/PartKeepr_1.4.0_installation_on_a_Raspberry_Pi](https://wiki.partkeepr.org/wiki/PartKeepr_1.4.0_installation_on_a_Raspberry_Pi)

[2] [https://github.com/partkeepr/PartKeepr/issues/905](https://github.com/partkeepr/PartKeepr/issues/905)