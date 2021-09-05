---
layout: post
title:  "ThingsBoard - MQTT"
date:   2019-01-13 12:30:00 +0800
categories: [IoT, MQTT]
tags: [MQTT, ThingsBoard]
---

## 1. 建立网关

设备 -> 添加新设备 -> 输入名称、类型，并勾选 `是网关`

## 2. 获取网关的访问令牌

设备 -> 点击添加的网关 -> 复制访问令牌

## 3. 上传测试数据

终端消息体示例：

```
"{"SN-002": [{"ts": 1548434563634,"values": {"temperature": 26.15,"humidity": 41.66,"voltage": 3.86,"rssi":-452,"status": 0}}]}"
```

上传测试数据：

```
mosquitto_pub.exe -d -h "127.0.0.1" -t "v1/gateway/telemetry" -m "{\"SN-002\": [{\"ts\": 1547355405,\"values\": {\"temperature\": 28,\"humidity\": 16,\"voltage\": 3.3,\"status\": 0}}]}" -u "[访问令牌]"
```

## 4. 远程过程调用

设备或网关订阅 RPC 的 Topic：

```
v1/devices/me/rpc/request/+
```

面板上添加的开关类型控件，默认会按如下格式向对应的设备发送控制消息：

```
topic: v1/devices/me/rpc/request/1
payload: {"method":"setValue","params":false}
```

如果在开关类型控件中配置了服务器要定时从设备获取状态，且获取方式为“Call RPC get value method”，服务器会向对应的设备发送如下消息：

```
topic: v1/devices/me/rpc/request/0
payload: {"method":"getValue"}
```

设备要使用下面的消息回应：

```
topic: v1/devices/me/rpc/response/0
payload: 暂时不知道，试试这个（{"value":true}）？
```



## 参考

[1] [https://thingsboard.io/docs/reference/gateway-mqtt-api/#publish-attribute-update-to-the-server](https://thingsboard.io/docs/reference/gateway-mqtt-api/#publish-attribute-update-to-the-server)

[2] [https://thingsboard.io/docs/reference/mqtt-api/](https://thingsboard.io/docs/reference/mqtt-api/)
