---
layout: post
title:  "嵌入式系统程序占用空间大小分析方法"
date:   2021-07-25 07:59:00 +0800
categories: [Embedded]
tags: [MCU]
excerpt: MCU 上的内存资源、flash 资源通常都比较紧张，有可能会因程序占用空间过大在编译、烧录、运行时出现莫名其妙的问题。本文以 STM8L151x4/6 和 Arduino UNO 为例，介绍 MCU 程序占用空间大小的分析方法。
---

## 1. 概述

MCU 上的内存资源、flash 资源通常都比较紧张，程序比较复杂的时候，有可能会出现程序 RAM 或 flash 占用空间过大，导致系统运行不稳定（比如栈和堆空间冲突导致程序出现莫名其妙或者复位的问题），甚至是大小超过硬件资源限制，导致编译、烧录失败的问题。

分析的基本方法大概有两种，一种是分析映射文件（\*.map)，统计各个段的大小，另一种是从最终生成的可执行可链接文件（\*.elf）中读取相关信息。

## 2. 查看整个程序的占用空间

### (1) STM8L151x4/6

以 STM8L151x4/6 为例，介绍第一种方法。

首先要获取映射文件。以 ST Visual Develop 为例，打开项目配置，在 `Linker` 选项卡中，将 `Category` 切换为 `Output`，在 `Map` 一栏勾选 `Generate Map File`，这样配置之后，编译时就可以生成映射文件了，例如 client.map。

下面的内容是 client.map 文件中的 `Segments` 部分。

```
                               --------
                               Segments
                               --------

start 00008080 end 0000820d length   397 segment .const
start 00008251 end 0000bfb7 length 15718 segment .text
start 00001000 end 00001000 length     0 segment .eeprom
start 00000000 end 00000000 length     0 segment .bsct
start 00000000 end 0000000a length    10 segment .ubsct
start 0000000a end 0000000a length     0 segment .bit
start 0000000a end 0000000a length     0 segment .share
start 00000100 end 0000013c length    60 segment .data, initialized
start 00008215 end 00008251 length    60 segment .data, from
start 0000013c end 0000052e length  1010 segment .bss
start 00000000 end 00002564 length  9572 segment .info.
start 00000000 end 00016f1d length 93981 segment .debug
start 00008000 end 00008080 length   128 segment .const
start 0000820d end 00008215 length     8 segment .init
```

先来看 flash 使用情况。STM8L151x4/6，从 0x8000 地址开始后的空间全部是 flash 空间（可以在芯片数据手册中找到），那么，从上面的内容可以看出，`.const` 段、`.text` 段、`.data` 段、`.init` 段占 flash 空间共计 16311 字节。`.const` 被分为两段是是因为 flash 的 0x8000~0x807f 是专门用来放复位和中断向量表的。

再来看 RAM 使用情况。STM8L151x4/6 的 RAM 地址段为 0x0000~0x07ff，所以，`.bsct` 段、`.ubsct` 段、`.bit` 段、`.share` 段、`.data` 段、`.bss` 段占 RAM 空间共计 1080 字节。

除了用来分析程序的占用空间，MAP 文件还可以有其他用途。例如，map 文件中 `Stack usage` 部分包含栈的使用信息：

```
                             -----------
                             Stack usage
                             -----------
// 省略
Stack size: 309
```

栈区位于 RAM 中（共 513 字节，栈底从 RAM 的截至地址 0x07ff 开始，即 0x05fe~0x07ff 为栈区）。

### (2) Arduino UNO

以 Arduino UNO 为例，介绍第二种方法。

首先要获取 ELF（Executable and Linking Format） 文件。打开 Arduino IDE 首选项设置，在 `Show verbose output during` 一行，勾选 `compile`，这样可以在从编译的输出中找到所需的所有信息，包括交叉编译工具链路径以及目标文件输出目录（编译完成后从该目录中找到后缀为 elf 的文件）。

接下来，使用交叉编译工具链提供的工具，从 elf 文件获取所需信息。

```
:: 查看各个段的大小
c:\>C:\Users\lenovo\AppData\Local\Arduino15\packages\arduino\tools\avr-gcc\7.3.0-atmel3.6.1-arduino7\bin\avr-size.exe -A C:\Users\lenovo\AppData\Local\Temp\arduino-sketch-D09F94A05940FB868995B43A5A7E8200\sketch_example1.ino.elf
:: 输出
C:\Users\lenovo\AppData\Local\Temp\arduino-sketch-D09F94A05940FB868995B43A5A7E8200\sketch_example1.ino.elf  :
section                     size      addr
.data                         22   8388864
.text                       4384         0
.bss                        1659   8388886
.comment                      17         0
.note.gnu.avr.deviceinfo      64         0
.debug_aranges               208         0
.debug_info                11541         0
.debug_abbrev               3127         0
.debug_line                 3898         0
.debug_frame                 820         0
.debug_str                  3076         0
.debug_loc                  5607         0
.debug_ranges                536         0
Total                      34959
```

首先来看一下 RAM 使用情况。`.data` 段存储的是<u>**初始化过**</u>的全局变量、<u>**初始化过**</u>的静态变量（包括全局静态变量和局部静态变量），共计 22 字节。`.bss` 段存储的是<u>**未初始化**</u>的全局变量、<u>**未初始化**</u>过的静态变量（包括全局静态变量和局部静态变量），二者加起来总共有 1681 字节，运行时，都会放到 RAM 中。按 Arduino UNO 的规格，RAM 有 2K，除去全局变量占用的 1681 字节，局部变量最多只能使用 367 字节。

接下来看一下 flash 使用情况。`.txt` 段存储的是程序执行代码，储存在 flash 中，有 4384 字节。除此以外，`.data` 段也会保存在 flash（`.bss` 段不占用 flash 空间）。`.txt` 和 `.data` 共占用 flash 中的 4406 字节。

从测试情况看，常量比较复杂，全局常量位于 `.data` 段。局部常量随着初值的不同，会影响 `.data` 段和 `.txt` 段。

各个段的含义可以参考这边博文：<<[Memory Layout of C Program](https://www.cs-fundamentals.com/c-programming/memory-layout-of-c-program-code-data-segments)>>。

## 3. 查看函数、变量、常量的占用空间

### (1) STM8L151x4/6

每个符号占用的空间可以在在映射文件的 `Symbols` 部分找到。

### (2) Arduino UNO

这部分参考了 <<[[Arduino] View code size and assembly code](https://blog.benoitblanchon.fr/arduino-code-size-and-assembly/)>>。

```
:: 按占用空间的大小从大到小打印符号名称
c:\>C:\Users\lenovo\AppData\Local\Arduino15\packages\arduino\tools\avr-gcc\7.3.0-atmel3.6.1-arduino7\bin\avr-nm.exe --size-sort -C -r C:\Users\lenovo\AppData\Local\Temp\arduino-sketch-D09F94A05940FB868995B43A5A7E8200\sketch_example1.ino.elf
:: 输出
000009a2 T main
00000400 b payloads_ring_buffer
000001cd b min_ctx
00000134 t on_wire_bytes(min_context*, unsigned char, unsigned char, unsigned char const*, unsigned int, unsigned int, unsigned char) [clone .constprop.14]
0000009d b Serial
0000009a t HardwareSerial::write(unsigned char)
00000094 T __vector_16
00000070 t stuffed_tx_byte(min_context*, unsigned char, bool) [clone .constprop.15]
00000064 T __vector_18
00000060 t crc32_step(crc32_context*, unsigned char)
0000005a t Print::write(unsigned char const*, unsigned int)
0000005a t _GLOBAL__sub_I___vector_18
0000004c t min_application_handler(unsigned char, unsigned char const*, unsigned char, unsigned char) [clone .constprop.8]
0000004c T __vector_19
00000048 t send_ack(min_context*) [clone .part.3] [clone .constprop.12]
00000048 t transport_fifo_send(min_context*, transport_frame*) [clone .constprop.13]
00000044 t HardwareSerial::_tx_udr_empty_irq()
00000040 t HardwareSerial::flush()
00000028 t HardwareSerial::read()
0000001e t HardwareSerial::availableForWrite()
0000001c t HardwareSerial::peek()
00000018 t millis
00000018 t HardwareSerial::available()
00000016 T __do_global_ctors
00000016 T __do_copy_data
00000014 t Serial0_available()
00000014 t serialEventRun()
00000012 d vtable for HardwareSerial
00000010 T __do_clear_bss
0000000c T __tablejump2__
00000004 b timer0_overflow_count
00000004 b timer0_millis
00000004 b last_sent
00000004 b now
00000001 b timer0_fract
```

## 4. 总结

两种分析程序占用空间的方法，一种是分析映射文件（\*.map)，统计各个段的大小，另一种是从最终生成的可执行可链接文件（\*.elf）中读取相关信息。第一种方法需要手动去统计，优点是结合数据手册分析会比较直观，缺点是比较麻烦。如果交叉编译工具链包含了解读二进制文件的工具，例如本文用到的 `avr-size`，使用第二种方法会比较方便。

## 参考

[1] [https://www.cs-fundamentals.com/c-programming/memory-layout-of-c-program-code-data-segments](https://www.cs-fundamentals.com/c-programming/memory-layout-of-c-program-code-data-segments)

[2] [https://blog.benoitblanchon.fr/arduino-code-size-and-assembly/](https://blog.benoitblanchon.fr/arduino-code-size-and-assembly/)