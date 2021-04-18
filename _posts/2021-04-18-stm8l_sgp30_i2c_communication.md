---
layout: post
title:  "STM8L SGP30 I2C 通信"
date:   2021-04-18 18:19:00 +0800
categories: [Embedded, MCU]
tags: [STM8]
---

最近使用 STM8L151K6T6 的硬件 I2C 接口和 SGP30 通信的过程中，遇到一个奇怪的问题：在发送完 "sgp30_iaq_int" 后的 15s 的传感器初始化时段内读取 CO2 和 TVOC 浓度没问题，但是 15s 之后会出现传感器不回应 Sr 之后的读请求的情况。一度以为 Sr 后发送的地址可能由于连线或者其他问题失真，传感器不应答。但是从逻辑分析仪的测量结果来看，地址是没有问题的。

![nack_after_read_req](/assets/img/2021-04-18-stm8l_sgp30_i2c_communication.assets/nack_after_read_req.png)

更奇怪的是，如果调试的时候，在发送 Sr 信号前添加断点，断住后再往下运行，传感器会回复 ACK，但是如果在发送 Sr 信号前添加延时，无论多久，传感器都不会回复 ACK。

仔细阅读传感器数据手册[[1]](https://www.sensirion.com/fileadmin/user_upload/customers/sensirion/Dokumente/9_Gas_Sensors/Datasheets/Sensirion_Gas_Sensors_Datasheet_SGP30.pdf)，6.2 节有这样的描述：

> A measurement communication sequence consists of a START condition, the I2C write header (7-bit I2C device address plus 0  as the write bit) and a 16-bit measurement command. The proper reception of each byte is indicated by the sensor. It pulls the  SDA pin low (ACK bit) after the falling edge of the 8th SCL clock to indicate the reception. With the acknowledgement of the  measurement command, the SGP30 starts measuring.

什么叫“the proper reception”呢？手册上没说，想来从接受命令到测量浓度也需要时间。但是，按照前面提到的，至少我遇到的这种情况下，延时多久都不行。

不过，这段内容后还有一些值得注意的内容：

> When the measurement is in progress, no communication with the sensor is possible and the sensor aborts the communication  with a XCK condition. 

这里说的 XCK 实际上就是 NACK，即 SDA 为高电平。按照这里的解释，读取数据的时候，测量还没结束，传感器回复了 XCK，让 SDA 保持高电平。那这样就能解释前面传感器没有回复 ACK 的问题了，由于 Read 信号 SDA 也是高电平，看起来好像传感器没有回复。

还有一段内容：

> After the sensor has completed the measurement, the master can read the measurement results by sending a START condition followed by an I2C read header.

传感器完成测量后，主设备需要发送 START 信号，然后是 I2C 读请求。就协议来说，这点没什么特别的，就是正常的 I2C 时序。结合前面的内容，既然第一次读的时候，传感器测量还没结束，返回了 XCK，那就多尝试几次 Sr+addr+Read 的动作。实测第二次就可以读到了：

![nack_after_read_req](/assets/img/2021-04-18-stm8l_sgp30_i2c_communication.assets/repeat_reading.png)

最开始提到的 15s 的初始化时间，在传感器数据手册的 6.3 节有说明。典型情况下，SGP30 在测量模式下消耗 48.8mA 电流，在低功耗的场景下，这 15s 的电流消耗也是不可忽略的存在。

至于断点调试时，传感器可以回复 ack 但延时却不行的问题，没想明白。

## 参考

[1] [https://www.sensirion.com/fileadmin/user_upload/customers/sensirion/Dokumente/9_Gas_Sensors/Datasheets/Sensirion_Gas_Sensors_Datasheet_SGP30.pdf](https://www.sensirion.com/fileadmin/user_upload/customers/sensirion/Dokumente/9_Gas_Sensors/Datasheets/Sensirion_Gas_Sensors_Datasheet_SGP30.pdf)

