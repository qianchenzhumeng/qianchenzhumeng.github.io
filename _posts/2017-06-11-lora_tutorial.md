---
layout: post
title:  "LoRa教程"
date:   2017-06-11 15:18:00 +0800
categories: [IoT, LoRa]
tags: [LoRa]
---

## 1. 概览

LoRa技术是一种广域网无线通信技术。基于LoRa无线通信技术的网络有多种不同的频段：902-928Hz（美国）、863-928Hz（欧盟）、779-787Hz（中国）等。LoRa是一种低功耗、长距离以及低通信速率的技术，最初由Samtech开发。

LoRa网络由网关、网络服务器以及终端设备组成，类型为星型。在LoRa网络系统中，终端设备也被称为motes，网关也被称为基站（base stations）或者集中器（concentratrators）。

终端和网关使用特定的ISM频段进行单跳连接。网关和网络服务器之间通过Ip回程线路连接（IP backhaul）。

![lora_network_architecture](/assets/img/2017-06-11-lora_tutorial.assets/lora_network_architecture.png)

上图描绘了LoRa网络的架构。客户信息数据库位于服务器中。终端设备和网关通过不同的信道和数据传输速率通信。LoRa支持的数据传输速率为0.3 Kbps - 50 Kbps。

## 2. LoRa 帧结构

从终端到网关的通信被称为上行链路（uplink），从网关到终端的通信被称为下行链路（downlink）。LoRaWAN支持不同类型的终端：A类，B类，C类。

![lora_classes](/assets/img/2017-06-11-lora_tutorial.assets/lora_classes.jpg)

如上图所示，LoRa帧由上行链路部分和下行链路部分组成。在Class A中，LoRa帧有一个上行链路槽，其后是两个下行链路槽。LoRa帧基于TDD技术。更多LoRa类别参见LoRa类别章节。

## 3. LoRa 协议栈

![lora_protocol_stack](/assets/img/2017-06-11-lora_tutorial.assets/lora_protocol_stack.jpg)

上图为Lora协议栈结构图。LoRa协议栈由应用层、介质访问控制层、物理层以及RF层组成。

- 被用来在终端和网关之间建立连接的、来自应用层的数据以及MAC命令，由MAC payload传输。
- MAC层利用MAC payload制作MAC帧。
- PHY层使用MAC作为PHY payload，并且在插入前导码、PHY报头、PHY报头循环冗余校验以及整个数据帧的循环冗余校验后用其制作PHY payload。
- RF层按照需要将PHY帧调制为需要的ISM Rf载波，并且将其发射到空中。

注意：本文中的内容取自2015年有LoRa联盟发布的《LoRaWAN Specification V1.0》。

## 4. LoRa 频段

LoRa无线系统在不同地区（例如：美国、欧盟、中国）使用不同的频段。下表列出了LoRa频段以及LoRa信道频率。需要注意的是，网关和终端都可以使用相同的频率配合不同的时间槽来传送数据，这个概念称为TDD。

| 地区 | LoRa频段        | LoRa信道频率                                                 |
| ---- | --------------- | ------------------------------------------------------------ |
| 欧盟 | 863 to  870 MHz | 868.10 Mhz (used  by Gateway to listen )   868.30 MHz (used  by Gateway to listen )  868.50 MHz (used  by Gateway to listen )   864.10 MHz (used  by End device to transmit Join Request)  864.30 MHz (used  by End device to transmit Join Request)  864.50 MHz (used  by End device to transmit Join Request)  868.10 MHz (used  by End device to transmit Join Request)  868.30 MHz (used  by End device to transmit Join Request)  868.50  MHz (used by End device to transmit Join Request) |
| 美国 | 902 to  928 MHz | 902.3 MHz to 914.9  MHz spaced at 200KHz (Upstream-64 channels)   903 MHz to 914.2  MHz spaced at 1.6 MHz apart (Upstream- 8 channels)   923.3  MHz to 927.5 MHz spaced at 600KHz apart (Downstream- 8 channels) |
| 中国 | 779 to  787 MHz | 779.5 MHz (Default  channel)   779.7 MHz (Default  channel)   779.9 MHz (Default  channel)   779.5 MHz (Used by  ED to transmit Join Request)  779.7 MHz (Used by  ED to transmit Join Request)  779.9 MHz (Used by  ED to transmit Join Request)  780.5 MHz (Used by  ED to transmit Join Request)  780.7 MHz (Used by  ED to transmit Join Request)  780.9  MHz (Used by ED to transmit Join Request) |

欧洲电信标准化协会（ETSI）为LoRa定义了433 MHz到434 MHz的频段，信道频率为433.175 MHz、433.375 Mhz以及433.575 MHz。在欧洲的频段中，Class B终端使用869.525 Mhz的信道。

## 参考

[1]https://www.rfwireless-world.com/Tutorials/LoRa-tutorial.html

[2] https://www.rfwireless-world.com/Tutorials/LoRa-frequency-bands.html