---
title: About
icon: fas fa-info
order: 4
---

 <center>
     <h1>关于博主</h1>
 </center>

&emsp;&emsp;工作经验
&emsp;&emsp;教育经历
&emsp;&emsp;技能
&emsp;&emsp;项目
&emsp;&emsp;荣誉
&emsp;&emsp;证书
&emsp;&emsp;个人账号
&emsp;&emsp;其他

## 工作经验

**软件开发工程师**

&emsp;&emsp;宏晶微电子科技股份有限公司

&emsp;&emsp;2021.08 ~ 至今

&emsp;&emsp;嵌入式软件开发

**驱动软件开发工程师**

&emsp;&emsp;中兴通讯股份有限公司

&emsp;&emsp;2018.03 ~ 2021.07

&emsp;&emsp;4G、5G基站、5G机载台以太网驱动软件开发，基带处理芯片驱动 SDK 开发

**产品研发工程师**

&emsp;&emsp;湖南浩盛消防科技有限公司

&emsp;&emsp;2016.07 ~ 2017.04

&emsp;&emsp;嵌入式软硬件、桌面应用软件开发

## 教育经历

**土木工程硕士**

&emsp;&emsp;2015.09 ~ 2018.06

&emsp;&emsp;中南大学

**消防工程学士**

&emsp;&emsp;2011.09 ~ 2015.06

&emsp;&emsp;中南大学

## 技能

* 精通 C 语言，熟悉 Rust，了解 Python、shell
* 熟悉基础数据结构和算法
* 熟悉 Linux 操作系统及其驱动软件开发
* 熟悉 Atomthreads 实时操作系统（为其添加了信号机制）
* 熟悉 STM32、STM8 软硬件开发
* 有工程制图基础，可熟练使用 FreeCAD，熟悉 3D 打印 

## 项目

- min-rs
  - 2021.07 ~ 2021.09
  - [MIN](https://github.com/min-protocol/min) 协议的 Rust 实现。
  - 项目地址：[https://github.com/qianchenzhumeng/min-rs](https://github.com/qianchenzhumeng/min-rs)
- IoT 网关
  - 2020.07 ~
  - 温室大棚温湿度检测系统的一部分（网关软件），使用 Rust 开发，运行在 OpenWrt 上，通过串口获取来自连接有 Lora 模块的单片机收到的数据，使用 MQTTS 将数据发送使用 ThingsBoard 搭建的服务器。终端节点使用 STM8L 系列单片机做主控，运行有 Atomthreads 实时操作系统。
  - 项目地址：[https://github.com/qianchenzhumeng/iot_gw](https://github.com/qianchenzhumeng/iot_gw)
- 基带处理芯片中台
  - 2020.11 ~ 2021.07
  - 第四代基带处理芯片基带板配套软件。参与芯片网络硬件加速器功能验证、SDK 和 ADK 开发，以及基站联调测试。
- 5G ATG
  - 2019.09 ~ 2019.11
  - 5G 机载台，给飞机提供互联网接入功能。前期牵头基带板调试工作，之后负责以太网驱动软件开发以及平台软件对业务软件的交付工作。
- 5G CPE
  - 2018.07 ~ 2019.09
  - 用于 5G 基站测试的设备。负责基带板以太网驱动软件开发以及平台软件对业务软件的交付工作。
- 基于Contiki的互联型独立式火灾探测报警器
  - 2017.05 ~ 2017.12
  - 通过自组织无线传感器网络实现独立式火灾探测报警器的互联，火灾探测报警器的主体为 STM32 单片机、烟雾探测报警模块、Sub1G 射频通信模块，运行有 Contiki 操作系统，通过改造的自组织按需向量路由协议实现节点间的自组网。
  - 项目地址：[https://github.com/qianchenzhumeng/FireDetectionAlarm](https://github.com/qianchenzhumeng/FireDetectionAlarm)
- 智能安全照明装置
  - 2016.07 ~ 2017.04
  - 集日常照明、火灾应急照明、烟雾报警于一体的智能安全照明装置。装置主体由单片机、红外感烟探测报警模块、wifi 模块、LED 模块构成，可以通过 airkiss 协议配网，检测到烟雾浓度过高时可以通过微信报警。
  - 实用新型专利：智能安全照明装置（专利号：2016100411351）
  - 实用新型专利：一种基于2.4G无线通信的自组网独立式火灾探测报警器（专利号：2016211093409）
  - 实用新型专利：智能安全照明装置状态指示灯（专利号：2016211093381）
  - 软件著作权：火灾报警照明灯固件嵌入式软件（登记号：2017SR248720）
  - 外观设计专利：烟雾探测照明灯（圆型）（专利号：2016300227706）
  - 外观设计专利：烟雾探测照明灯（方型）（专利号：2016300227674）
  - 外观设计专利：烟雾探测照明灯（古韵型）（专利号：2016300227693）
  - 外观设计专利：烟雾探测照明灯（古雅型）（专利号：2016300227710）
  - 项目地址（软件中的一部分）：[https://github.com/qianchenzhumeng/ESP8266-STM32F103](https://github.com/qianchenzhumeng/ESP8266-STM32F103)
- 基于 Zigbee 的火灾模拟试验数据采集系统
  - 2014.10 ~ 2015.06
  - 采集火灾模拟实验中的烟气流速数据，终端节点将数据通过 Zigbee 传给数据收集器实时显示、处理。终端节点主控为 CC2530，数据收集器主体为 Beagle Bone Black 开发板，运行 Linux 操作系统，应用软件使用 QT 开发。
  - 软件著作权：火灾模拟试验数据采集系统（登记号：2015SR152625）
  - 实用新型专利：基于Zigbee的火灾模拟实验数据采集系统（专利号：2015205485193）
- 消防水泵运行状况自动采集监控系统
  - 2014.04 ~ 2014.05
  - 监控消防水泵的启停状态。设备由单片机、加速度传感器、GPRS模块组成，服务器软件采用 node.js 开发
  - 实用新型专利：一种消防水泵运行状况的自动采集监控系统（实用新型专利：201420818800X）
- 火灾烟气流速数据采集系统
  - 2013.10 ~ 2014.01
  - 采集火灾模拟实验中的烟气流速数据。设备由单片机、微压差传感器、毕托管构成，上位机软件使用 QT 开发
  - 软件著作权：火灾烟气流速数据采集系统（登记号：2014SR046435）

## 荣誉

* 中南大学2015年优秀毕业生
* 中南大学防灾科学与安全技术研究所2014年“创意之星”

## 证书

- Coursera Certificate
  - [Java 程序设计：使用软件解题](https://www.coursera.org/account/accomplishments/certificate/UFEESK5Y37WB)
  - [Embedded Hardware and Operating Systems](https://www.coursera.org/account/accomplishments/certificate/P2KRR4PZ6DWY)
  - [Machine Learning](https://www.coursera.org/account/accomplishments/certificate/BXN9Q8SUXN5H)
- 全国计算机等级考试四级（网络工程师）合格证书
- 全国计算机等级考试二级（C）合格证书

## 个人账号

* 博客
  * [https://www.kiloleaf.com](https://www.kiloleaf.com)
  * [https://qianchenzhumeng.github.io/](https://qianchenzhumeng.github.io/)
* GitHub
  * [https://github.com/qianchenzhumeng](https://github.com/qianchenzhumeng)
* Tingsiverse
  * [https://www.thingiverse.com/mikeyu/designs](https://www.thingiverse.com/mikeyu/designs)
* 邮箱
  * [qianchenzhumeng@live.cn](javascript:window.open('mailto:' + ['qianchenzhumeng','live.cn'].join('@')))

## 其他

* 喜欢骑车、看书、钻研技术
* 性格内敛、沉稳