---
layout: post
title:  "用 Rust 实现 MIN 协议"
date:   2021-08-29 09:29:00 +0800
categories: [Rust, Other]
tags: [Rust]
excerpt: 写这篇文章的目的是介绍一下MIN协议，并且对使用Rust实现MIN协议的一些考量做个分享。
---

## 1. 背景

[MIN](https://github.com/min-protocol/min/wiki) 协议，全称 Microcontroller Interconnect Network protocol，是为微控制器和微控制器或者微控制器和 PC 设计的点对点传输协议，工作在串口上面，可发送任意数据，支持校验、重传机制。

之前一直想做一个 Lora 网关，尝试过树莓派和 Lora 模块组合的形式，树莓派上跑 Eclipse 的 Kura 网关，通过 SPI 接口控制、访问 Lora 模块。Lora 模块上没有 CPU，直接就是 Lora 射频收发芯片加外围电路，在 Kura 框架下，用 java 操作 SPI 还是比较难受的，而且来自 Lora 模块的中断也比较难处理，再加上树莓派跑 Kura 比较费劲，后来就放弃这条路了。

之后买了现成的 Lora 网关 Dragino LG01-P，这个网关和 Arduino Yun 的架构类似，Linux + MCU + Lora 模块的架构，Linux 上面跑 OpenWrt，MCU 侧对外体现为 Arduino UNO R3，Lora 模块由 MCU 直接控制，OpenWrt 和 MCU 通过串口通讯。这种架构是比较好的，一来 OpenWrt 有 Web 页面，二来编程时可以避免在 Linux 用户态进行比较低级的操作，访问串口要比操作 SPI、接收中断方便。但是软件就不太方便了。MCU 和 OpenWrt 的交互大概是这样的：MCU 的串口连接在 OpenWrt 的控制台上，交互由 MCU 发起，就像人在 Linux 控制台上命令一样，MCU 通过串口将标准的 Linux 命令（例如，cat a.txt）发送给 OpenWrt，控制台运行完命令后，MCU 可以接收到执行结果。LG01-P 提供的 MQTT 传输就受到了这方面的影响。上传的时候，MCU 把数据整理成字符串，通过 echo 命令存到 OpenWrt 上的某个文件里。OpenWrt 侧有定时任务，该任务会调用一个 shell 脚本，这个脚本会调用 mosquitto 把数据发布出去。这才是上行。下行的话，数据要从服务器到 OpenWrt，经过 MCU，由 Lora 模块发送出去，在这种模式下，更难实现。除此之外，MCU 资源有限，处理字符串力不从心，实际使用中很容易出现栈和堆空间冲突程序崩溃的情况。

为了解决上面的这个问题，我把 OpenWrt 的控制台从和 MCU 相连的那个串口断开了，然后用 Rust 写了个网关程序 [iot_gw](https://github.com/qianchenzhumeng/iot_gw)，在 OpenWrt 上监听和 MCU 连的那个串口，收到数据后会对数据进行一系列转换、封装，然后发送给服务器。OpenWrt 和 MCU 使用自己设计的 [HDTP](https://github.com/qianchenzhumeng/iot_gw/tree/master/hdtp) 协议传输数据。这样的话，MCU 只需要传输数据即可，比起传输 Linux 命令要节省资源，而且下行也比较容易实现。

[HDTP](https://github.com/qianchenzhumeng/iot_gw/tree/master/hdtp) 协议比较简单，用来传输可见字符，设计上没有转义、重传、可靠传输之类的考量。一开始想用 [MIN](https://github.com/min-protocol/min/wiki) 协议，但是当时该协议没有 Rust 实现，而且从文档也看不出具体的通信过程，就借鉴 HDLC 协议设计并实现了 [HDTP](https://github.com/qianchenzhumeng/iot_gw/tree/master/hdtp) 协议。网关侧实现了接收功能，MCU 上实现了简单的数据帧封装，也就是说只实现了上行发送，还不支持下行接收。

上行其实是为了把传感器采集到的信息发送到服务器上，按照计划，接下来需要开发控制功能，即服务器把命令下发给底层的执行机构，这样的话，网关和 MCU 都需要实现上行接收、下行发送功能。[MIN](https://github.com/min-protocol/min/wiki) 已经有了 Arduino 的实现，加上这一年来也遇到并解决了 [HDTP](https://github.com/qianchenzhumeng/iot_gw/tree/master/hdtp) 不少的 Bug，这个时候就不打算继续重复造轮子了，仔细研究了 [MIN](https://github.com/min-protocol/min/wiki) 的源码，用 Rust 实现了一遍，后续打算用 [MIN](https://github.com/min-protocol/min/wiki) 替换掉 [HDTP](https://github.com/qianchenzhumeng/iot_gw/tree/master/hdtp)。网关程序使用 Rust 实现的 [min-rs](https://github.com/qianchenzhumeng/min-rs)，MCU 上使用原开发者实现的 Arduino 程序。

## 2. MIN 协议介绍

### (1) 协议构成

[MIN](https://github.com/min-protocol/min/wiki) 协议由数据帧和传输协议两部分构成。数据帧最大可以传输 255 字节的有效数据。传输协议是可选的，如果开启，可以提供应答、重传机制。

### (2) 帧格式

```
+------------+------------+------------+------------+------------+------------+------------+-----------+----------+------------+------------+------------+------------+------------+------------+
|    SOF 0   |    SOF 1   |    SOF 2   | ID/Control | (Sequence) |  Length    | Data 0     | Data 1    |   ...    | Data 255   |  Checksum  |  Checksum  |  Checksum  |  Checksum  |    EOF     |
|    0xAA    |    0xAA    |    0xAA    |            |            |            |            |           |          |            |   (MSB)    |            |            |   (LSB)    |    0x55    |
+------------+------------+------------+------------+------------+------------+------------+-----------+----------+------------+------------+------------+------------+------------+------------+
```

- SOF 域：Start Of Frame，长度为 3 个字节，固定为 0xAAAAAA。
- ID/Control 域：长度为 1 个字节。BIT7 表明该数据帧是传输帧（启用了传输协议）还是非传输帧（没有启用传输协议），BIT6 为保留位，必须是 0 ，BIT0 - BIT5 留给上层应用使用。
- Sequence 域：长度为 1 个字节，用于传输协议，非传输帧中没有该位域。
- Length 域：长度为 1 个字节，有效数据长度。
- Data 域：有效数据，长度由 Length 域指定。原始数据如果含有 SOF 序列（连续 3 字节的 0xAA），协议会对其进行转义处理：如果是发送，每两字节的 0xAA 后会插入 1 字节 0x55；对应的，如果是接收，两字节 0xAA 后的转义字符会被丢弃。
- Checksum 域：4 个字节 CRC-32 校验码。
- EOF 域：End Of Frame，长度为 1 个字节，固定为 0x55。

### (3) C 代码实现

主机端的程序是 C 语言写的，大体上按是否启用传输协议分为两部分，使用 `TRANSPORT_PROTOCOL` 宏区分。功能方面，读写串口的接口以及给上层应用传输数据的接口都是以回调的形式存在的。

先说不启用传输协议的情况。上层应用发送数据时直接调用 `min_send_frame` 接口，该接口会通过 `on_wire_bytes` 函数将数据封装为非传输帧，调用串口发送接口发出去。接收方面采用状态机，串口接收函数每次收到数据，都需要调用 `min_poll` 函数，将收到的数据送给状态机。状态机收到完整的数据后，对数据进行 CRC 校验，如果校验通过则通过回调函数将数据发送给上层应用。

接下来是启用传输协议的情况。这种情况下，程序需要周期性调用 `min_poll` 来驱动协议栈工作。上层应用发送数据时调用 `min_queue_frame` 接口，该接口会将数据存入一个环形缓冲区，协议栈工作时会从环形缓冲区拷贝数据帧发送，发送过程有应答和重传机制，确认对端已经收到后，会将该帧数据从环形缓冲区移除。接收数据的方式和不启用数据传输协议时的是一样的，不过 `min_poll` 中有额外的处理流程，包括检查时间窗、重传、应答等。

### (4) 未启用传输协议时的通信过程

不启用传输协议的通信过程很简单，发送方按如下帧格式（ID/Control 的 BIT7 为 1，不含 Sequence 域）发送数据即可：

```
+------------+------------+------------+------------+------------+------------+-----------+----------+------------+------------+------------+------------+------------+------------+
|    SOF 0   |    SOF 1   |    SOF 2   | ID/Control |  Length    | Data 0     | Data 1    |   ...    | Data 255   |  Checksum  |  Checksum  |  Checksum  |  Checksum  |    EOF     |
|    0xAA    |    0xAA    |    0xAA    |            |            |            |           |          |            |   (MSB)    |            |            |   (LSB)    |    0x55    |
+------------+------------+------------+------------+------------+------------+-----------+----------+------------+------------+------------+------------+------------+------------+
```

### (5) 启用传输协议时的通信过程

1. 发送方发送数据帧（包含 Sequence 域）
2. 接收发自己维护计数器 rn，其值为希望收到的下一帧的序列号。收到数据帧时，如果 rn 和报文携带的序列号相等，rn 自增 1，并且用新的 rn 值发送确认帧（向对端表明对端在发送下一帧数据帧时需要携带该序列号），并将数据传给上层应用，否则丢弃该数据帧。
3. 发送方收到确认帧时，释放对应的报文。

确认帧（ACK）格式如下：

```
+------------+------------+------------+------------+------------+------------+------------+------------+------------+------------+------------+------------+
|    SOF 0   |    SOF 1   |    SOF 2   | ID/control |  Sequence  |  Length    | Data 0     |  Checksum  |  Checksum  |  Checksum  |  Checksum  |    EOF     |
|    0xAA    |    0xAA    |    0xAA    |    0xFF    |     rn     |     1      |     rn     |   (MSB)    |            |            |   (LSB)    |    0x55    |
+------------+------------+------------+------------+------------+------------+------------+------------+------------+------------+------------+------------+
```

### (6) 实例分析（启用传输协议）

1. 分析 A 向 B 发送一帧传输协议帧数据，然后线路空闲一段之间，之后 B 向 A 发送一帧传输协议帧数据的情况

```
A                               B
|         Frame(seq=0)--------->|   A 发送完数据帧后，会把 sn_max 加 1
|        ACK(seq=0,rn=0)------->|
|<-------ACK(seq=1,rn=1)        |   B 发送确认帧，A 接收到确认帧后，会把 sn_min 更新为 seq
|        ACK(seq=0,rn=0)------->|
|              ...              |   线路空闲
|<--------Frame(seq=0)          |   B 发送完数据帧后，会把 sn_max 加 1
|<-------ACK(seq=1,rn=1)        |
|        ACK(seq=1,rn=1)------->|   A 发送确认帧，B 接收到确认帧后，会把 sn_min 更新为 seq
|<-------ACK(seq=1,rn=1)        |
|              ...              |   线路空闲
```

2. 分析 A -> B 发送一帧数据帧，该帧数据帧丢失的情况

```
A                               B
|         Frame(seq=0)----X-----|   A 发送完数据帧后，会把 sn_max 加 1
|        ACK(seq=0,rn=0)------->|
|<-------ACK(seq=0,rn=0)        |
|         Frame(seq=0)--------->|   A 会不断重试，直到收到来自 B 的 ACK(seq=1,rn=1)
|        ACK(seq=0,rn=0)------->|
|<-------ACK(seq=1,rn=1)        |   B 发送确认帧，A 接收到确认帧后，会把 sn_min 更新为 seq，并且释放报文
|        ACK(seq=0,rn=0)------->|
               ...
|              ...              |   线路空闲
```

3. 分析 A -> B 发送多帧数据帧，且这些数据帧全部丢失的情况，重传后 B 才收到的情况

```
A                               B
|         Frame(seq=0)----X-----|   A 发送完数据帧后，会把 sn_max 加 1
|        ACK(seq=0,rn=0)------->|
|         Frame(seq=1)----X-----|   A 发送完数据帧后，会把 sn_max 加 1
|        ACK(seq=0,rn=0)------->|
|         Frame(seq=2)----X-----|   A 发送完数据帧后，会把 sn_max 加 1
|        ACK(seq=0,rn=0)------->|
               ...
|         Frame(seq=0)----X---->|   A 不断重试从 0 到 2 逐个发送
|        ACK(seq=0,rn=0)------->|
|         Frame(seq=1)----X---->|
|        ACK(seq=0,rn=0)------->|
|         Frame(seq=2)----X---->|
|        ACK(seq=0,rn=0)------->|
               ...
|         Frame(seq=0)--------->|
|        ACK(seq=0,rn=0)------->|
|<-------ACK(seq=1,rn=1)        |   B 发送确认帧，A 接收到确认帧后，会把 sn_min 更新为 seq，并且释放报文（seq=0）
|         Frame(seq=1)--------->|
|        ACK(seq=0,rn=0)------->|
|<-------ACK(seq=2,rn=2)        |   B 发送确认帧，A 接收到确认帧后，会把 sn_min 更新为 seq，并且释放报文（seq=1）
|         Frame(seq=2)--------->|
|        ACK(seq=0,rn=0)------->|
|<-------ACK(seq=3,rn=3)        |   B 发送确认帧，A 接收到确认帧后，会把 sn_min 更新为 seq，并且释放报文（seq=2）
|              ...              |   线路空闲
```

## 3. 使用 Rust 实现 MIN 协议

使用 Rust 实现 MIN 协议的主要挑战在于回调函数的处理，由于所有权以及生命周期的限制，不能在回调函数中像 C 语言那样通过以全局变量形式存在的串口句柄、应用程序句柄来发送、接收数据。

解决办法是将串口及应用程序的句柄，在初始化的时候以不可变引用的形式在传递给 MIN 的句柄。这个串口的句柄需要单独抽象出来，需要可变借用的时候，为其实现内部可变性，避免引发可变借用及不可变借用冲突的问题。

为了便于调试，例如，打印应用程序及串口标识以进行区分。实现时采用了这样的方法：串口名称的打印，在串口发送、接收函数中打印即可。应用程序的句柄需要实现一个名为 `Name` 的 Trait，这个 Trait 会返回一个字符串，在 MIN 的处理过程中可以通过应用程序句柄的这个 Trait 返回应用程序的名称。

源代码放在 GitHub 上（[min-rs](https://github.com/qianchenzhumeng/min-rs)）。

## 4. 外部链接

[1] [MIN protocol](https://github.com/min-protocol/min)

[2] [MIN protocol wiki](https://github.com/min-protocol/min/wiki)

[3] [min-rs：MIN 协议的 Rust 实现 ](https://github.com/min-protocol/min)

[4] [iot_gw：用 Rust 编写的 IOT 网关](https://github.com/qianchenzhumeng/iot_gw)