---
layout: post
title:  "Mosquitto 笔记"
date:   2016-03-29 00:09:00 +0000
categories: [IoT, MQTT]
tags: [MQTT, ESP8266]
---

## 1. MQTT

MQTT（MQ Telemetry Transport）是轻量级的消息传输协议，它允许设备高效地通过受限网络与远程系统进行异步通信。

MQTT的特点

- 发布/订阅模式可以提供一对多的消息发布，解除了应用程序之间的耦合

- 消息传输屏蔽了报文的内容

- 使用TCP/IP提供基本的网络连接

- 三种用于消息分发的质量服务：

- - “至多一次”
  - “至少一次”
  - “只有一次”

- 传输开销很小（固定长度的报头只有两字节），协议交换最小化，以减少网络流量

- 使用Last Will和Testament特性来通知相关部分客户端异常断开链接的机制

## 2. Mosquitto

Mosquitto 是一个开源的消息代理软件，支持 MQTT V3.1 和 V3.1.1 协议。

## 3. Raspberry Pi 安装 Mosquitto

```
$ sudo apt-get install libssl-dev libc-ares-dev uuid-dev
$ wget http://mosquitto.org/files/source/mosquitto-1.4.8.tar.gz
$ tar xvzf mosquitto-1.4.8.tar.gz
$ cd mosquitto-1.4.8
$ make
$ sudo make install
$ sudo ln -s /usr/local/lib/libmosquitto.so.1 /usr/lib/libmosquitto.so.1
$ sudo ldconfig
```

## 4. Centos 安装 Mosquitto

```
# wget http://mosquitto.org/files/source/mosquitto-1.4.8.tar.gz
# tar xvzf mosquitto-1.4.8.tar.gz
# cd mosquitto-1.4.8
# make
# make install
# ln -s /usr/local/lib/libmosquitto.so.1 /usr/lib/libmosquitto.so.1
# ldconfig
```

## 5. 测试

一个完整的MQTT示例包括代理器、发布者和订阅者。这里打开三个终端，分别代表代理器、发布者和订阅者。

(1) 启动 Mosquitto 服务器（代理器）

```
mosquitto -v
```

(2) 订阅主题（订阅者）

```
mosquitto_sub -v -t sensor
```

(3)发布内容（发布者）

```
mosquitto_pub -t sensor -m 12
```

当发布者推送消息后，订阅者会获得以下内容：

> sensor 12

## 6. 可能遇到的问题

(1) make[1]: cc: Command not     found

```
# yum install gcc
```

(2) error: ares.h: No such file     or directory

```
# yum install c-ares-devel
```

(3) make[2]: g++: Command not     found

```
# yum install gcc-c++
```

(4) error: uuid/uuid.h: No such     file or directory

```
# yum install libuuid-devel
```

## 7. 其他

需要注意的是，Centos 上的软件包和 debian 上的软件包命名规则不同。编译过程中如果提示缺少某个头文件，可以用 `yum search [filename]` 命令搜索一下，找到相应的软件包再安装。