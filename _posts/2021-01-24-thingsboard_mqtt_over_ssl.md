---
layout: post
title:  "ThingsBoard MQTT over SSL"
date:   2021-01-24 22:37:00 +0800
categories: [IoT, Server]
tags: [ThingsBoard]
excerpt: 这篇文章介绍了如何在开源 IoT 平台 ThingsBoard 上启用 mqtts。
---

## 1. 背景

本文使用 ThingsBoard 3.1 版本，设备通过 MQTTS 接入该版本时，可以选择 SSL 单向认证或双向认证。

使用 SSL 单向认证时，客户端使用服务端的证书来校验服务端。客户端通过这种方式接入 ThingsBoard 时，还需要提供自己的接入令牌。证可以使用 ThingsBoard 源码中提供的脚本来生成，客户端的接入令牌在 ThingsBoard 的设备管理界面设置。

使用 SSL 双向认证时，客户端和服务端相互认证，客户端使用服务端的证书来校验服务端，服务端使用客户端的证书来校验客户端。证书可以使用 ThingsBoard 的源码生成。

ThingsBoard 可以同时支持 MQTT 和 MQTTS，即，资源够的设备可以使用 MQTTS 接入，资源受限的设备可以通过 MQTT 接入。

## 2. 为客户端和服务端生成自签证书

ThingsBoard 源码中有生成 SSL 证书的脚本，使用这些脚本可以很方便地生成有效期是 9999 天的服务端和客户端需要的证书文件（私钥、公钥对）。

从 [ThingsBoard 仓库](https://github.com/thingsboard/thingsboard/blob/release-3.1/tools/src/main/shell) 下载这三个文件：

- client.keygen.sh
- keygen.properties
- server.keygen.sh

其中，“keygen.properties” 的内容如下：

```shell
#
# Copyright © 2016-2017 The Thingsboard Authors
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#

DOMAIN_SUFFIX="$(hostname)"
ORGANIZATIONAL_UNIT=Thingsboard
ORGANIZATION=Thingsboard
CITY=SF
STATE_OR_PROVINCE=CA
TWO_LETTER_COUNTRY_CODE=US

SERVER_KEYSTORE_PASSWORD=server_ks_password
SERVER_KEY_PASSWORD=server_key_password

SERVER_KEY_ALIAS="serveralias"
SERVER_FILE_PREFIX="mqttserver"
SERVER_KEYSTORE_DIR="/etc/thingsboard/conf"

CLIENT_KEYSTORE_PASSWORD=password
CLIENT_KEY_PASSWORD=password

CLIENT_KEY_ALIAS="clientalias"
CLIENT_FILE_PREFIX="mqttclient"
```

生成证书前，需要修改 “keygen.properties” 文件中的部分，其中，`$(hostname)` 务必替换为 IP 地址或域名，其余的根据需要修改，比如：

```shell
DOMAIN_SUFFIX="www.domainname.com"              # 域名
ORGANIZATIONAL_UNIT=domainname
ORGANIZATION=domainname
CITY=XIAN
STATE_OR_PROVINCE=SHANNXI
TWO_LETTER_COUNTRY_CODE=CN

SERVER_KEYSTORE_PASSWORD=server_ks_password     # KeyStore 文件密码
SERVER_KEY_PASSWORD=server_key_password         # 服务端证书密码

SERVER_KEY_ALIAS="serveralias"
SERVER_FILE_PREFIX="mqttserver"                 # 生成的服务端证书相关文件的文件名
SERVER_KEYSTORE_DIR="./"                        # 服务端证书相关文件的输出路径

CLIENT_KEYSTORE_PASSWORD=client_ks_password     # 客户端 KeyStore 文件密码
CLIENT_KEY_PASSWORD=client_key_password         # 客户端证书密码

CLIENT_KEY_ALIAS="mqttclientalias"
CLIENT_FILE_PREFIX="mqttclient"                 # 生成的客户端证书相关文件的文件名
```

先要运行 “server.keygen.sh”，生成服务器相关的证书：

|        文件        |                      说明                       |
| :----------------: | :---------------------------------------------: |
|   mqttserver.cer   |                服务端的证书文件                 |
|   mqttserver.jks   |              服务端 KeyStore 文件               |
| mqttserver.pub.pem | 纯文本格式的文件，内容是 PEM 格式的服务端的证书 |

然后运行 “client.keygen.sh”，生成客户端相关的证书：

|         文件          |                             说明                             |
| :-------------------: | :----------------------------------------------------------: |
|    mqttclient.jks     |                     客户端 KeyStore 文件                     |
| mqttclient.nopass.pem | 纯文本格式的文件，内容是 PEM 格式的客户端的 RSA 私钥和客户端的证书（RSA 公钥） |
|    mqttclient.p12     |                      客户端 PKCS12 文件                      |
|    mqttclient.pem     | 纯文本格式的文件，内容是 PEM 格式的客户端的 PKCS12 密钥和客户端的证书（RSA 公钥） |
|  mqttclient.pub.pem   | 纯文本格式的文件，内容是 PEM 格式的客户端的证书（RSA 公钥）  |

RSA 私钥存放在 `*.nopass.pem` 文件中，以“-----BEGIN RSA PRIVATE KEY-----”开头，以“-----END RSA PRIVATE KEY-----”结尾。

RSA 密钥转换而来的 PKCS12 密钥存放在 `*.pem` 文件中，以“-----BEGIN ENCRYPTED PRIVATE KEY-----”开头，以“-----END ENCRYPTED PRIVATE KEY-----”结尾。

证书在 `*.nopass.pem`、`*.pem` 和 `*.pub.pem` 中都有（内容一样），以“-----BEGIN CERTIFICATE-----”开头，以“-----END CERTIFICATE-----”结尾。

这些证书文件及脚本要保存好，后续添加新设备时会用到：

- 将 `mqttserver.cer`、`mqttserver.jks`、`keygen.properties` 及 `client.keygen.sh` 复制到同一目录下；
- 必要时，修改 `keygen.properties` 中 `CLIENT_FILE_PREFIX` 的值，比如改为新设备的名称；
- 运行 `client.keygen.sh` 为新设备生成证书、密钥等。

## 3. ThingsBoard 启用 MQTTS 配置

### (1) Linux

将服务端的 KeyStore 文件复制到 ThingsBoard 的配置文件目录下，并修改所有者、权限：

```shell
sudo cp mqttserver.jks /etc/thingsboard/conf/
sudo chmod 400 /etc/thingsboard/conf/mqttserver.jks
sudo chown thingsboard:thingsboard /etc/thingsboard/conf/mqttserver.jks
```

关于启用 MQTTS 的修改，可以修改配置文件目录下的“thingsboard.yml”或“thingsboard.conf”，二者选一就行。

如果修改“thingsboard.yml”，需要注意以下几行：

```
sudo vim /etc/thingsboard/conf/thingsboard.yml
```

```shell
mqtt:
  enabled: "${MQTT_ENABLED:true}"
  bind_address: "${MQTT_BIND_ADDRESS:0.0.0.0}"
  bind_port: "${MQTT_BIND_PORT:1883}"
  ssl:
    enabled: "${MQTT_SSL_ENABLED:true}"                                     # 使能 MQTTS
    bind_address: "${MQTT_SSL_BIND_ADDRESS:0.0.0.0}"
    bind_port: "${MQTT_SSL_BIND_PORT:8883}"                                 # MQTTS 端口号
    protocol: "${MQTT_SSL_PROTOCOL:TLSv1.2}"                                # SSL 协议版本
    credentials:
      type: "${MQTT_SSL_CREDENTIALS_TYPE:KEYSTORE}"                         # 使用 KeyStore 类型的证书文件
      keystore:
        type: "${MQTT_SSL_KEY_STORE_TYPE:JKS}"
        store_file: "${MQTT_SSL_KEY_STORE:mqttserver.jks}"                  # 服务端 KeyStore 文件名称
        store_password: "${MQTT_SSL_KEY_STORE_PASSWORD:server_ks_password}" # KeyStore 文件密码
        key_alias: "${MQTT_SSL_KEY_ALIAS:serveralias}"                      # 服务端密钥别名
        key_password: "${MQTT_SSL_KEY_PASSWORD:server_key_password}"        # 服务端密钥密码
    skip_validity_check_for_client_cert: "${MQTT_SSL_SKIP_VALIDITY_CHECK_FOR_CLIENT_CERT:false}"
```

如果要修改“thingsboard.conf”，在里面添加一些环境变量：

```shell
sudo vim /etc/thingsboard/conf/thingsboard.conf
```

```shell
export MQTT_SSL_ENABLED=true
export MQTT_SSL_KEY_STORE_TYPE=KEYSTORE
export MQTT_BIND_PORT=8883
export MQTT_SSL_KEY_STORE=mqttserver.jks
export MQTT_SSL_KEY_STORE_PASSWORD=server_ks_password
export MQTT_SSL_KEY_PASSWORD=server_key_password
```

### (2) Windows

打开配置文件 `thingsboard.yml`（例如：`D:\Program Files (x86)\thingsboard-windows-3.1\thingsboard\conf\thingsboard.yml`），参照 linux 上 MQTTS 的设置修改相应的内容。

服务端 KeyStore 文件需要放置在 `conf` 目录下，例如`D:\Program Files (x86)\thingsboard-windows-3.1\thingsboard\conf\mqttserver.jks`。

## 4. 服务器及客户端配置

### (1) 单向认证

先在 ThingsBoard 的设备管理页面新增一个设备，默认的凭据类型是 `Access token`，不需要修改，记录下访问令牌的内容即可，比如：“EyufOOJOx8QetxbopMc2”。

以使用 mosquitto_pub 接入 ThingsBoard 为例：

```bash
mosquitto_pub -d -q 1 -h "localhost" -p "8883" -t "v1/devices/me/telemetry" -u "EyufOOJOx8QetxbopMc2" -m {"temperature":25} --cafile "mqttserver.pub.pem"
```

### (2) 双向认证

#### 1) 服务端配置

先要将客户端的公钥添加到服务器上。登录 ThingsBoard 的设备管理页面，将需要进行双向认证的设备的凭据类型修改为 `X.509 Certificate`，并将“mqttclient.pub.pem”的内容（设备的证书）复制到 `RSA 公钥` 区域。

#### 2) 客户端配置

设置好服务端后，客户端与服务端通过 MQTTS 连接时，需要指定 CA 文件（服务器的 PEM 格式的证书）、客户端的 PEM 格式的证书（公钥）、客户端的 PEM 格式的私钥。

如果使用 mosquitto_pub，通过 `--cafile` 指定 PEM 格式的 CA 文件，通过 `--cert` 指定客户端的 PEM 格式的公钥文件，通过 `--key` 指定客户端的 PEM 格式的私钥文件：

```shell
mosquitto_pub -d -h "localhost" -p 8883 -t "v1/devices/me/telemetry" -m '{"temperature":42}' --cafile "mqttserver.pub.pem" --cert "mqttclient.nopass.pem" --key "mqttclient.nopass.pem" --tls-version "tlsv1.2"
```

“mqttclient.nopass.pem” 同时包含了客户端地公钥和私钥，因此 `--cert` 和 `--key` 都可以指定这个文件（即，也可以把私钥和公钥从这个文件中复制出来，单独保存，单独指定）。

如果使用 paho.mqtt.rust，通过如下代码指定服务端 CA 文件，以及客户端的私钥、公钥：

```rust
let ssl_opts = mqtt::SslOptionsBuilder::new()
    .trust_store("mqttserver.pub.pem")
    .key_store("mqttclient.nopass.pem")
    .finalize();
```

## 5. 可能遇到的问题及解决方案

生成客户端的证书时，如果遇到如下报错：

```
keytool error: java.security.UnrecoverableKeyException: Get Key failed: Given final block not properly padded. Such issues can arise if a bad key is used during decryption.
```

则将 `keygen.properties` 中 `SERVER_KEY_PASSWORD` 的值修改为 `SERVER_KEYSTORE_PASSWORD` 的值，然后在生成客户端的证书。

单向认证时，日志里可以看到如下内容：

```
WARN  o.t.s.t.mqtt.MqttTransportHandler - peer not authenticated
```

这个没有影响，服务器是可以收到来自设备的数据的。

收不到消息时，可能看到如下信息：

```
INFO  o.t.s.s.q.DefaultTbRuleEngineConsumerService - Timeout to process [2] messages
```

重启了一下 ThingsBoard 好了。

## 参考

[1] [https://thingsboard.io/docs/user-guide/mqtt-over-ssl/](https://thingsboard.io/docs/user-guide/mqtt-over-ssl/)

[2] [https://thingsboard.io/docs/user-guide/certificates/](https://thingsboard.io/docs/user-guide/certificates/)

[3] https://tutorialspedia.com/an-overview-of-one-way-ssl-and-two-way-ssl/