---
layout: post
title:  "MQTT Embeded 学习笔记（订阅消息）"
date:   2016-04-27 10:09:00 +0000
categories: [IoT, MQTT]
tags: [MQTT, ESP8266]
---

MQTTSNPacket/samples文件夹下有向服务器订阅消息的例程，如"pub0sub1.c"。下面仍旧以ST-NUCLEO-F103RB和ESP8266为例，说一下使用MQTTPacket如何向服务器订阅消息（未提到的部分请参阅《MQTT Embedded学习笔记（发布）》）。

## 1. 例程解读

"pub0sub1.c"中main()函数主要有以下几部分：

(1) 发送CONNECT命令
主要的代码为：

```
len = MQTTSerialize_connect(buf, buflen, &data);
```

这行代码将含有CONNECT命令的信息串行化后放在buf里面。

(2) 接收CONNACK

主要代码为：

```
if (MQTTPacket_read(buf, buflen, transport_getdata) == CONNACK)
{
	unsigned char sessionPresent, connack_rc;
	if (MQTTDeserialize_connack(&sessionPresent, &connack_rc, buf, buflen) != 1 || connack_rc != 0)
	{
		printf("Unable to connect, return code %d\n", connack_rc);
		goto exit;
	}
}
else
	goto exit;
```

其中，`MQTTPacket_read(buf, buflen, transport_getdata)` 通过 `transport_getdata` 函数读取接收到的信息，提取出消息类型，并将 payload 放进 buf 中。`MQTTDeserialize_connack(&sessionPresent, &connack_rc, buf, buflen)` 函数将接收到的数据进行反串行化，以确认是否连接成功。该段代码中 `transport_getdata` 函数原型如下（定义在 `MQTTSNPacket/samples/transport.c` 文件中）：

```
int transport_getdata(unsigned char* buf, int count);
```

该函数的功能是从缓冲区读取 count 个数据放入 buf 中，缓冲区类似于 FIFO 的队列，`transport_getdata` 函数实际上是对缓冲区中的前 count 个元素进行出队操作。同之前（MQTT Embedded学习笔记（发布））的 `Socket_new` 函数一样，`transport_getdata` 函数及 FIFO 的缓冲区都需要自己构造。

(3) 订阅消息SUBSCRIBE

主要代码如下：

```
len = MQTTSerialize_subscribe(buf, buflen, 0, msgid, 1, &topicString, &req_qos);
```
这行代码将包含订阅信息的数据进行串行化，并将其储存在 buf 中。

(4) 接收SUBACK

主要代码如下：

```
if (MQTTPacket_read(buf, buflen, transport_getdata) == SUBACK) 	/* wait for suback */
{
	unsigned short submsgid;
	int subcount;
	int granted_qos;
	rc = MQTTDeserialize_suback(&submsgid, 1, &subcount, &granted_qos, buf, buflen);
	if (granted_qos != 0)
	{
		printf("granted qos != 0, %d\n", granted_qos);
		goto exit;
	}
}
else
	goto exit;
```

其中，`MQTTDeserialize_suback` 函数对接收到的数据进行反串行化，检出质量服务等级等信息。

(5) 接收服务器发布的消息

主要代码如下：

```
if (MQTTPacket_read(buf, buflen, transport_getdata) == PUBLISH)
{
	unsigned char dup;
	int qos;
	unsigned char retained;
	unsigned short msgid;
	int payloadlen_in;
	unsigned char* payload_in;
	int rc;
	MQTTString receivedTopic;
	rc = MQTTDeserialize_publish(&dup, &qos, &retained, &msgid, &receivedTopic,
				&payload_in, &payloadlen_in, buf, buflen);
	printf("message arrived %.*s\n", payloadlen_in, payload_in);
}
```

其中，`MQTTPacket_read()` 函数从数据中提取出消息类型。`MQTTDeserialize_publish()` 函数将接收到的数据进行反串行化，检出接收到的消息数据。

(6) 发布消息

请参阅《MQTT Embedded学习笔记（发布）》。

## 2. 编码

接下来自己构造合适的函数：

(1) int transport_getdata(unsigned char* buf, int count);

这里将函数改为：

```
int tcp_getdata(unsigned char* buf, int count)
{
	int i;
	if(count <= TCP_RX_BUFFER_LEN)
	{
		for(i = 0; i < count; i++)
		{
			*(buf + i) = tcpRxBuffer[i];
		}
		for(i = 0; i < TCP_RX_BUFFER_LEN - count; i++)
		{
			tcpRxBuffer[i] = tcpRxBuffer[i+count];
		}
		return count;
	}
	else
	{
		return -1;
	}
}
```

其中，tcpRxBuffer 类似于 FIFO 队列，出队操作在此处完成，入队操作在另外的函数中完成。

(2) bool esp8266ReadTcpData()

```
bool esp8266ReadTcpData()
{
	bool rc;
	int len = 0;
	uint8_t i, byteOfLen = 0;
	char *p;
	rc = esp8266ReadForResponse(RESPONSE_RECEIVED, COMMAND_RESPONSE_TIMEOUT);
	if(rc)
	{
		p = strstr((const char *)esp8266RxBuffer, "+IPD,");
		if(p)
		{
			p += 5;
			while(*p != ':')
			{
				if(byteOfLen == 0)
					len += (int)(*p - '0');
				else if(byteOfLen == 1)
					len = (int)(*p - '0') + len * 10;
				else if(byteOfLen == 2)
					len += (int)(*p - '0') + len * 10;
				p++;
				if(byteOfLen >= 3)	//数据接收不完整，跳出循环（接下来的4个字节内没有接收到冒号）。
				{
					byteOfLen = 0;
					return FALSE;
				}
				byteOfLen++;
			}
			p++;
			for(i = 0; i < len; i++)
			{
				tcpRxBuffer[i] = *p;
				p++;
			}
			esp8266ClearBuffer();
			return TRUE;
		}
		else
			return FALSE;
	}
	else
	{
		return FALSE;
	}
}
```

该函数从串口接收缓存数据中检出来自 TCP 套接字的数据，需要结合 ESP8266 的 AT 指令集编写。这里假设接收到的数据不会大于999（三位）个字节。

不足之处在于，需要采用查询的方式及时从串口接收缓存中提取出来自 TCP 套接字的数据，否则串口接收缓存中的数据会被来自 ESP8266 的最新的数据（不一定包含来自 TCP 套接字的数据）覆盖。一种思路是在串口接收中断中使用状态机机制，及时将来自TCP套接字的数据提取出来。

(3) 其他函数

包括创建 TCP 套接字、发送数据、关闭 TCP 套接字等（请参阅《MQTT Embedded学习笔记（发布）》）。
