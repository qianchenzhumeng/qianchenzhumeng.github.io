---
layout: post
title:  "Arduino存储器"
date:   2017-08-29 09:09:00 +0800
categories: [Embedded, MCU]
tags: [IoT, Arduino]
---

Arduino有三种类型的存储器：

- Flash or Program Memory
- SRAM（Static Random     Access Memory）
- EEPROM（Electrically     Erasable Programmable Read and Write Memory）

## (1) Flash Memory 

Flash用于储存程序镜像以及初始化数据。Flash是非易失性存储器，因此，系统掉电后，程序依然储存在里面。Flash的擦写寿命大概为10万次。

## (2) SRAM 

SRAM可以在程序运行时读写，常被用于以下目的：

- Static Data：这个区块用于储存程序中所有的全局变量以及静态变量。对于有初始值的变量，当程序运行时，系统会从Flash中取出初始值。
- Heap：用于动态分配内存。当分配动态内存时，堆从静态数据区域的顶部向上增长。
- Stack：用于储存局部变量、维护中断记录以及函数调用记录。栈从SRAM的顶部开始，并向堆的方向增长。每次中断时，函数调用或者局部变量的分配会引起栈增长。从中断或函数调用中返回时，用于中断或函数的栈空间会被释放。

![1](/assets/img/2017-08-29-arduino_memories.assets/1.png)

绝大多数内存问题发生在栈和堆空间冲突时。这种情况发生时，一方或者两者的内存空间会受到破坏，导致不可预测的结果。在一些情况下，这会导致程序立刻崩溃；在另外一些情况下，内存空间受到破坏造成的影响可能在稍迟一些才会显现。

![2](/assets/img/2017-08-29-arduino_memories.assets/2.png)

## (3) EEPROM

EEPROM是另一种非易失性存储器，可在程序运行时读取。它只能被逐字节读取，并且读写速度要比SRAM慢，擦写寿命大概为10万次。

## 参考

[1] [https://learn.adafruit.com/memories-of-an-arduino/arduino-memories](https://learn.adafruit.com/memories-of-an-arduino/arduino-memories)