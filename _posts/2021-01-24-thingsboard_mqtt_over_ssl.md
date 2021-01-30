---
layout: post
title:  "ThingsBoard MQTT over SSL"
date:   2021-01-24 22:37:00 +0800
categories: [IoT, Server]
tags: [ThingsBoard]

---

## 1. 背景

ThingsBoard 版本：3.1

## 2. 后台配置

### (1) Linux

生成自签证书

从 [ThingsBoard仓库](https://github.com/thingsboard/thingsboard/tree/master/tools/src/main/shell) 下载这三个文件：

- client.keygen.sh
- keygen.properties
- server.keygen.sh

修改 `keygen.properties` 中的内容，务必将 `$(hostname)` 替换为 ip 或域名，其余的根据需要修改：

```
# DOMAIN_SUFFIX="$(hostname)"
DOMAIN_SUFFIX="localhost"
```

依次运行 `server.keygen.sh` 和 `client.keygen.sh`，生成 server 和 client 的密钥：

```shell
sh server.keygen.sh
sh client.keygen.sh
```

将 server 密钥库复制到 ThingsBoard 的配置文件目录下，并修改所有者、权限：

```shell
sudo cp mqttserver.jks /etc/thingsboard/conf/
sudo chmod 400 /etc/thingsboard/conf/mqttserver.jks
sudo chown thingsboard:thingsboard /etc/thingsboard/conf/mqttserver.jks
```

在配置文件中添加内容：

```shell
sudo vim /etc/thingsboard/conf/thingsboard.conf
```

如果修改了 `keygen.properties` 中服务器密钥和密钥库的密码的话，记得替换 `server_ks_password` 和 `server_key_password`：

```shell
export MQTT_SSL_ENABLED=true
export MQTT_BIND_PORT=8883
export MQTT_SSL_KEY_STORE=mqttserver.jks
export MQTT_SSL_KEY_STORE_PASSWORD=server_ks_password
export MQTT_SSL_KEY_PASSWORD=server_key_password
```

### (2) Windows

打开配置文件 `thingsboard.yml`（例如：`D:\Program Files (x86)\thingsboard-windows-3.1\thingsboard\conf\thingsboard.yml`），参照 linux 上 MQTTS 的设置修改相应的内容。

密钥库需要放置在`conf`目录下，例如`D:\Program Files (x86)\thingsboard-windows-3.1\thingsboard\conf\mqttserver.jks`。

## 3. 服务器及客户端配置

### (1) 服务器

需要修改设备凭据类型：

设备 -> DETAILS -> 管理凭据：将凭据类型设置为 `X.509 Certificate`，将 mqttclient.pub.pem 文件内容复制到 `RSA 公钥` 区域。

### (2) 客户端

mosquitto_pub：

```shell
mosquitto_pub -d -h "localhost" -p 8883 -t "v1/devices/me/telemetry" -m '{"temperature":42}' --cafile "mqttserver.pub.pem" --cert "mqttclient.nopass.pem" --key "mqttclient.nopass.pem" --tls-version "tlsv1.2"
```

paho.mqtt.rust：

```rust
let ssl_opts = mqtt::SslOptionsBuilder::new()
    .trust_store("mqttserver.pub.pem")
    .key_store("mqttclient.nopass.pem")
    .finalize();
```

## 参考

[1] https://thingsboard.io/docs/user-guide/mqtt-over-ssl/

[2] https://thingsboard.io/docs/user-guide/certificates/