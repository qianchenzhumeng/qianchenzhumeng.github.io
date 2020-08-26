---
layout: post
title:  "MQTT Embeded 学习笔记（发布消息）"
date:   2016-03-30 15:45:00 +0800
categories: [IoT, MQTT]
tags: [MQTT, STM32, ESP8266]
---

## 1. Paho

Eclipse 的 Paho 项目旨在提供可伸缩的开放和标准的 Machine-to-Machine (M2M) 以及物联网消息协议的开源实现。Paho 提供了许多不同版本的 MQTT client 以供不同平台使用。其中，Embedded MQTT C/C++ Client Libraries 是为嵌入式平台提供的，可以将其用在 mbed、Arduino、FreeRTOS 等环境中。

## 2. Embedded MQTT C/C++ Client Libraries

Embedded MQTT C/C++ Client Libraries包含三部分内容：MQTTPacket、MQTTClient和MQTTClient-C。

- MQTTPacket
- MQTTClient
- MQTTClient-C

MQTTPacket是最底层的库，最小最简单，但是也相对难用。它仅仅对MQTT数据包进行了串行化和反串行化处理。串行化是指处理了应用程序的数据并将数据转换成便于通过网络传输的格式。反串行化是指处理了从网络中接收到的数据并将数据提取了出来。

MQTTClient是一个C++库，是为mbed写的，但是现在也可以用在其他平台上。尽管它使用了C++，但是它避免了动态内存分配，并且对于操作系统和网络依赖函数有可替换的类。也避免了标准模板库（STL）的使用。MQTTClient基于MQTTPacket，并且需要用到MQTTPacket。

MQTTClient-C是MQTTClient的C语言版，用于不能使用C++的环境，例如FreeRTOS。也是基于MQTTPacket的。

## 3. 发布消息

下载MQTT Embedded：

```
git clone https://git.eclipse.org/r/paho/org.eclipse.paho.mqtt.embedded-c
```

将MQTTSNPacket/src文件夹下的文件复制到项目代码文件夹内。

Eclipse的官网给了一段MQTTClient和一段MQTTPacket的示例代码。代码不长，使用起来还是很容易的。下面以 ST-NUCLEO-F103RB 和 ESP8266 为例，说一下MQTTPacket的使用。

官网示例代码如下：

```
MQTTPacket_connectData data = MQTTPacket_connectData_initializer;
int rc = 0;
char buf[200];
MQTTString topicString = MQTTString_initializer;
char* payload = "mypayload";
int payloadlen = strlen(payload);int buflen = sizeof(buf);

data.clientID.cstring = "me";
data.keepAliveInterval = 20;
data.cleansession = 1;
len = MQTTSerialize_connect(buf, buflen, &data); /* 1 */

topicString.cstring = "mytopic";
len += MQTTSerialize_publish(buf + len, buflen - len, 0, 0, 0, 0, topicString, payload, payloadlen); /* 2 */

len += MQTTSerialize_disconnect(buf + len, buflen - len); /* 3 */

rc = Socket_new("127.0.0.1", 1883, &mysock);
rc = write(mysock, buf, len);
rc = close(mysock);
```

上述代码需要包含头文件“MQTTPacket.h ”，使用时还需要将“MQTTPacket”文件夹中的文件添加到工程当中。以上代码完成这样的事情：将客户端命名为“me”，向位于本地（127.0.0.0:1883）的代理器发布一段消息，消息的topci叫“mytopic”，内容为“mypayload”，质量服务（qos）等级为0，即“至多一次”。

使用只需要根据实际情况对上述代码做部分改动就行，例如“clientID”、"topicString"、"payload"以及最后三行的函数。最后的三个函数依次是创建套接字、将串行化好的数据包发送出去、关闭套接字。下面写三个函数来实现这三个功能。

这里假设ESP8266的驱动已经写好。

创建套接字：

```
bool socketNew(uint8_t* host, uint8_t * port)
{
	bool rc = esp8266TcpConnect(host, port);
	return rc;
}
```

`esp8266TcpConnect(host, port)` 的函数原型是 `bool esp8266TcpConnect(uint8_t * destination, uint8_t * port)`，该函数使用 `AT+CIPSTART` 指令来创建 TCP 连接。`bool` 是个自定义的枚举类型，定义如下：

```
typedef enum
{
	FALSE = 0,
	TRUE = 1
}bool;
```

发送数据：

```
bool socketWrite(uint8_t* pBuf, uint16_t buflen)
{
	bool rc = esp8266TcpSend(pBuf, buflen);
	return rc;
}
```

`esp8266TcpSend(pBuf, buflen)` 的函数原型是 `bool esp8266TcpSend(uint8_t *buf, uint16_t size)`，该函数使用 `AT+CIPSEND` 指令来发送数据。

关闭套接字：

```
bool socketClose(void)
{
	bool rc = esp8266TcpClose();
	return rc;
}
```

`esp8266TcpClose()` 的函数原型是 `bool esp8266TcpSend(uint8_t *buf, uint16_t size)`，该函数使用 `AT+CIPCLOSE` 指令来关闭 TCP 连接。

测试时，可以在电脑上安装 MQTT 服务器，将 ESP8266 连入和电脑相同的网络。如果要在 Linux 上安装，可以参考《Mosquitto学习笔记》。
