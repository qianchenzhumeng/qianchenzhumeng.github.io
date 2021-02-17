---
layout: post
title:  "ThingsBoard3.1 安装"
date:   2021-01-30 20:58:00 +0800
categories: [IoT, Server]
tags: [ThingsBoard]

---

## 1. Linux

### (1) 安装 ThingsBoard 服务

```shell
sudo dpkg -i thingsboard-3.1.deb
```

### (2) 安装 java 8

```shell
sudo apt install openjdk-8-jdk
sudo update-alternatives --config java
```

### (3) 安装 PostgreSQL

```shell
# install **wget** if not already installed:
sudo apt install -y wget

# import the repository signing key:
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -

# add repository contents to your system:
RELEASE=$(lsb_release -cs)
echo "deb http://apt.postgresql.org/pub/repos/apt/ ${RELEASE}"-pgdg main | sudo tee  /etc/apt/sources.list.d/pgdg.list

# install and launch the postgresql service:
sudo apt update
sudo apt -y install postgresql-12
sudo service postgresql start
```

Once PostgreSQL is installed you may want to create a new user or set the password for the the main user.  The instructions below will help to set the password for main postgresql user

```shell
sudo su - postgres
psql
\password
\q
```

Then, press “Ctrl+D” to return to main user console and connect to the database to create thingsboard DB:

```shell
psql -U postgres -d postgres -h 127.0.0.1 -W
CREATE DATABASE thingsboard;
\q
```

### (4) 配置数据库配置

修改 ThingsBoard 配置文件

```shell
sudo vim /etc/thingsboard/conf/thingsboard.conf
```

在配置文件中添加以下内容，注意将 `PUT_YOUR_POSTGRESQL_PASSWORD_HERE` 替换为 postgres 的用户密码密码：

```shell
# DB Configuration 
export DATABASE_ENTITIES_TYPE=sql
export DATABASE_TS_TYPE=sql
export SPRING_JPA_DATABASE_PLATFORM=org.hibernate.dialect.PostgreSQLDialect
export SPRING_DRIVER_CLASS_NAME=org.postgresql.Driver
export SPRING_DATASOURCE_URL=jdbc:postgresql://localhost:5432/thingsboard
export SPRING_DATASOURCE_USERNAME=postgres
export SPRING_DATASOURCE_PASSWORD=PUT_YOUR_POSTGRESQL_PASSWORD_HERE
export SPRING_DATASOURCE_MAXIMUM_POOL_SIZE=5
# Specify partitioning size for timestamp key-value storage. Allowed values: DAYS, MONTHS, YEARS, INDEFINITE.
export SQL_POSTGRES_TS_KV_PARTITIONING=MONTHS
```

### (5) 运行安装脚本

Once ThingsBoard service is installed and DB configuration is updated, you can execute the following script:

```
# --loadDemo option will load demo data: users, devices, assets, rules, widgets.
sudo /usr/share/thingsboard/bin/install/install.sh --loadDemo
```

### (6) 启动 ThingsBoard 服务

```shell
sudo service thingsboard start
```

## Windows

thingsboard-windows-3.1 和 postgresql-11.10-2-windows-x64 是配套的，使用 postgresql-13.1-1-windows-x64，会导致 thingsboard 安装失败：

> org.postgresql.util.PSQLException: 不支援 10 验证类型。请核对您已经组态 pg_hba.c
> onf 文件包含客户端的IP位址或网路区段，以及驱动程序所支援的验证架构模式已被支援。

启动：

```
net start thingsboard
```

停止：

```
net stop thingsboard
```

## 其他

(1) 默认账号

- System Administrator: sysadmin@thingsboard.org / sysadmin
- Tenant Administrator: tenant@thingsboard.org / tenant
- Customer User: customer@thingsboard.org / customer
- Customer User: customerA@thingsboard.org / customer
- Customer User: customerB@thingsboard.org / customer
- Customer User: customerC@thingsboard.org / customer

(2) 启停相关命令

linux

```shell
# 检查是否开机启动
systemctl is-enabled thingsboard
# 开机启动 thingsboard
sudo systemctl enable thingsboard
# 禁止开机启动 thingsboard
sudo systemctl disable thingsboard
```

windows

在 **服务** 中查找`ThingsBoard Server Application`、`postgresql-x64-11 - PostgreSQL Server 11`。

(3) 删除数据库

linux

```bash
sudo su - postgres
psql -U postgres -d postgres -h 127.0.0.1 -W
DROP DATABASE thingsboard;
\q
```

windows

打开 pgAdmin 4，在浏览器里面删除。

(4) 卸载

ubuntu

```shell
sudo apt-get remove --purge thingsboard
```

windows

以管理员权限打开 CMD，运行安装目录下的 `uninstall.bat`。

## 参考

[1] [https://thingsboard.io/docs/user-guide/install/linux/](https://thingsboard.io/docs/user-guide/install/linux/)

[2] [https://thingsboard.io/docs/user-guide/install/windows/](https://thingsboard.io/docs/user-guide/install/windows/)