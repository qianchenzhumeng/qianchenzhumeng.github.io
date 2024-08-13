---
layout: post
title:  "Keil HEX 文件格式解析及数据提取"
date:   2022-02-13 21:15:00 +0800
categories: [Embedded, MCU]
tags: [Keil]
excerpt: 本文以 MCU OTA 升级为背景，对 Intel HEX 文件格式以及 Keil 输出内存数据的方式进行简单介绍，并提供 Intel HEX 文件以及 BIN 文件数据提取示例程序。
---

## 1. 背景

进行 MCU OTA 升级调试时，升级重启后，MCU 程序跑飞，怀疑程序下载缓冲区的内容被改写，为了排除这个疑点，需要将缓冲区内的数据与编译生成的 BIN 文件或 HEX 文件内的数据进行比对。思路是将数据以文本形式输出到文件中，使用文件比对工具进行比对。

Keil 的 `SAVE` 调试命令会将下载缓冲区中的数据输出到 Intel HEX 格式[[1]](https://developer.arm.com/documentation/ka003292/latest)的文件中，虽然编译生成的 `*.hex` 文件也是该格式，但是二者有时不能直接对比：前者的地址会从 `0x0000` 开始，但是编译生成的 `*.hex` 文件内记录的起始地址与实际指定的程序烧录地址有关，但是数据是一样的。因此，需要从这两个文件中分别提取数据进行比对。如果是 BIN 文件，仅包含二进制数据，可以将数据提取出来，以同样的文本形式输出到文件中进行比对。

## 2. 文件格式

### (1) Intel HEX 文件格式

Keil 生成的 Intel HEX 文件中，每一行都是一条 HEX 记录，形如：

```
:llaaaatt[dd...]cc
```

每条记录的格式如下[[1]](https://developer.arm.com/documentation/ka003292/latest)：

- `:` 每条记录都以冒号开始.
- `ll` 是“记录长度”字段，表示该条记录中“数据”字段（dd）占多少字节。
- `aaaa` 是“地址”字段，表示该条记录中，“数据”字段所在的起始地址。
- `tt` 是“类型”字段，表示该条记录的类型，有如下值：
   **00** - 数据（data）
   **01** - 文件结束（end-of-file）
   **02** - 扩展段地址（extended segment address）
   **04** - 扩展线性地址（extended linear address）
   **05** - 开始线性地址（start linear address），仅 MDK-ARM 使用
- `dd` 是“数据”字段，表示一个字节的数据。一条记录中可能有多个字节的数据，长度由 `ll` 字段指定。
- `cc` 是记录的校验和。

Keil 输出的 HEX 文件，每条记录最多包含 16 字节的数据，例如：

```
:10100000D0BC01208D420008978300084D70000875
```

### (2) BIN 文件格式

BIN 文件其实没有格式。

## 3. 数据提取示例代码

### (1) 提取 HEX 文件内的数据

接下来给出一段提取 HEX 文件内容的程序，该程序逐行读取 HEX 文件，如果遇到类型为“数据”的记录，则将该记录中的“数据”字段提取出来，以十六进制 ASCII 字符的形式输出到另一个文件中。

```c
/**
 * @file hex_tool.c
 * @author qianchenzhumeng (qianchenzhumeng@live.cn)
 * @brief 从 Intel HEX 文件中提取数据，以 ASCII 文本的形式输出到指定文件中。
 * @version 0.1
 * @date 2022-02-12
 * 
 * @copyright Copyright (c) 2022 qianchenzhumeng
 * 
 */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdint.h>

/* Inter HEX 文件格式描述：
 *  : 每条记录都以冒号开始.
 *  ll 是“记录长度”字段，表示该条记录中“数据”字段（dd）占多少字节。
 *  aaaa`是“地址”字段，表示该条记录中，“数据”字段所在的起始地址。
 *  tt 是“类型”字段，表示该条记录的类型，有如下值：
 *    00 - 数据（data）
 *    01 - 文件结束（end-of-file）
 *    02 - 扩展段地址（extended segment address）
 *    04 - 扩展线性地址（extended linear address）
 *    05 - 开始线性地址（start linear address），仅 MDK-ARM 使用
 *  dd 是“数据”字段，表示一个字节的数据。一条记录中可能有多个字节的数据，长度由 `ll` 字段指定。
 *  cc 是记录的校验和。
 */

/**
 * @brief 记录类型
 * 
 */
typedef enum xRECORD_TYPE
{
    eData,
    eEOF,
    eExtendedSegmentAddress,
    eExtendedLineAddress,
    eStartLinearAddress
}xRecordType_t;

/**
 * @brief 行信息
 * 
 */
typedef struct xHEX_80_LINE_INFO
{
    /*  LL AAAA TT DDDD CC*/
    /* :02 0000 04 2000 DA */
    /*  LL AAAA TT DDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDD CC*/
    /* :10 1000 00 D0BC01208D420008978300084D700008 75 */
    uint8_t u8Length;
    uint16_t u16Address;
    uint8_t u8RecordType;
    uint8_t u8CheckSum;
}xHexLineInfo_t;

#define LINE_MAX_LENGTH (1024)
char acLineData[LINE_MAX_LENGTH];

/**
 * @brief 解析 HEX 行数据，要保证传给该函数的行数据是有效的 HEX 行数据，否则可能发生错误
 * 
 * @param[in] pcLineData 行数据
 * @param[out] pxHexLineInfo 行信息结构体地址
 * 
 * @return 数据开始的地址
 */
void *pvHexLineParser(char *pcLineData, xHexLineInfo_t *pxHexLineInfo)
{
    char acString[5];

    if( (NULL == pcLineData) || (NULL == pxHexLineInfo) )
    {
        return NULL;
    }

    /* 获取 LL 字段 */
    memset((void *)acString, '\0', sizeof(acString));
    memcpy((void *)acString, (void *)(pcLineData + 1), 2);
    pxHexLineInfo->u8Length = (uint8_t)strtol(acString, NULL, 16);

    /* 获取 AAAA 字段 */
    memset((void *)acString, '\0', sizeof(acString));
    memcpy((void *)acString, (void *)(pcLineData + 3), 4);
    pxHexLineInfo->u16Address = (uint16_t)strtol(acString, NULL, 16);

    /* 获取 TT 字段 */
    memset((void *)acString, '\0', sizeof(acString));
    memcpy((void *)acString, (void *)(pcLineData + 7), 2);
    pxHexLineInfo->u8RecordType = (uint8_t)strtol(acString, NULL, 16);

    /* 获取 CC 字段 */
    memset((void *)acString, '\0', sizeof(acString));
    memcpy((void *)acString, (void *)(pcLineData + 9 + pxHexLineInfo->u8Length * 2), 2);
    pxHexLineInfo->u8CheckSum = (uint8_t)strtol(acString, NULL, 16);

    return pcLineData+9;
}

void vPrintHelp(char *pcAppName)
{
    if( NULL != pcAppName)
    {
        printf("Usage: %s [input file] [output file]\n", pcAppName);
    }
}

int main(int argc, char *argv[])
{
    FILE *fInput;
    FILE *fOutput;
    size_t xLines = 0;
    size_t xLineLength = 0;
    size_t xFileSize;
    char cr;
    char *pcInputFileName = NULL;
    char *pcOutputFileName = NULL;
    char *pcData = NULL;
    xHexLineInfo_t xHexLineInfo;

    if( 3 != argc )
    {
        vPrintHelp(argv[0]);
        exit(1);
    }
    else
    {
        pcInputFileName = argv[1];
        pcOutputFileName = argv[2];
        printf("output: %s\n", pcOutputFileName);
    }

    fInput = fopen(pcInputFileName, "rb"); 
    if (fInput == NULL)
    {
        printf("Open %s error!\n", pcInputFileName);
        exit(1);
    }
    fOutput = fopen(pcOutputFileName, "wb+");
    if( NULL == fOutput )
    {
        fclose(fInput);
        printf("Open %s error!\n", pcOutputFileName);
        exit(1);
    }

    /* 获取文件大小 */
    fseek(fInput, 0, SEEK_END);
    xFileSize = ftell(fInput);
    rewind(fInput);

    /* 统计输入文件行数 */
    for(size_t i = 0; i < xFileSize; i++)
    {
        cr = getc(fInput);
        if( '\n' == cr)
        {
            xLines++;
        }
    };
    rewind(fInput);

    /* 解析行信息，提取数据，以 ASCII 文本的形式储存到文件中 */
    for(size_t i = 0; i < xLines; i++)
    {
        fgets(acLineData, LINE_MAX_LENGTH, fInput);
        pcData = pvHexLineParser(acLineData, &xHexLineInfo);
        if( NULL != pcData )
        {
            if( eData == xHexLineInfo.u8RecordType )
            {
                fwrite(pcData, 1 , xHexLineInfo.u8Length*2, fOutput);
                fwrite("\n", 1, 1, fOutput);
            }
        }
    }

    fclose(fInput);
    fclose(fOutput);

    return 0;
}
```

例如，将 [[1]](https://developer.arm.com/documentation/ka003292/latest) 中给出的示例文件的如下内容保存在 `test.hex` 文件中：

```
:10001300AC12AD13AE10AF1112002F8E0E8F0F2244
:10000300E50B250DF509E50A350CF5081200132259
:03000000020023D8
:0C002300787FE4F6D8FD7581130200031D
:10002F00EFF88DF0A4FFEDC5F0CEA42EFEEC88F016
:04003F00A42EFE22CB
:00000001FF
```

将前述代码编译为二进制文件，例如 `hex_tool.exe`，使用该程序提取数据，输出到 `test.txt` 文件中：

```bash
./hex_tool.exe test.hex test.txt
```

`test.txt` 文件中的内容如下：

```
AC12AD13AE10AF1112002F8E0E8F0F22
E50B250DF509E50A350CF50812001322
020023
787FE4F6D8FD758113020003
EFF88DF0A4FFEDC5F0CEA42EFEEC88F0
A42EFE22
```

### (2) 提取 BIN 文件内的数据

下面的这段代码，是从编译生成的 BIN 文件中读取数据，按每行 16 个十六进制字符的形式输出到指定的文件中：

```c
/**
 * @file bin_tool.c
 * @author qianchenzhumeng (qianchenzhumeng@live.cn)
 * @brief 读取 BIN 文件内容，以 ASCII 文本的形式输出到指定文件中。
 * @version 0.1
 * @date 2022-02-12
 * 
 * @copyright Copyright (c) 2022 qianchenzhumeng
 * 
 */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdint.h>

void vPrintHelp(char *pcAppName)
{
    if( NULL != pcAppName)
    {
        printf("Usage: %s [input file] [output file]\n", pcAppName);
    }
}

int main(int argc, char *argv[])
{
    FILE *fInput;
    FILE *fOutput;
    char cr = '\0';
    uint8_t aucChar[3] = {0, 0, 0};
    uint8_t i = 0;
    size_t xFileSize;

    char *pcInputFileName = NULL;
    char *pcOutputFileName = NULL;
    char *pcData = NULL;

    if( 3 != argc )
    {
        vPrintHelp(argv[0]);
        exit(1);
    }
    else
    {
        pcInputFileName = argv[1];
        pcOutputFileName = argv[2];
        printf("output: %s\n", pcOutputFileName);
    }

    fInput = fopen(pcInputFileName, "rb"); 
    if (fInput == NULL)
    {
        printf("Open %s error!\n", pcInputFileName);
        exit(1);
    }
    fOutput = fopen(pcOutputFileName, "wb+");
    if( NULL == fOutput )
    {
        fclose(fInput);
        printf("Open %s error!\n", pcOutputFileName);
        exit(1);
    }

    /* 获取文件大小 */
    fseek(fInput, 0, SEEK_END);
    xFileSize = ftell(fInput);
    rewind(fInput);

    for(size_t j = 0; j < xFileSize; j++)
    {
        cr = getc(fInput);
        snprintf(aucChar, sizeof(aucChar),"%02X", (uint8_t)cr);
        fwrite(aucChar, 2, 1, fOutput);
        //printf("%02X", (uint8_t)cr);
        if( (0 != i)  && (i == 15) )
        {
            fwrite("\n", 1, 1, fOutput);
            //printf("\n");
            i = 0;
        }
        else
        {
            i++;
        }
    }
    fwrite("\n", 1, 1, fOutput);

    fclose(fInput);
    fclose(fOutput);

    return 0;
}
```

将前述代码编译为二进制文件，例如 `bin_tool.exe`，使用该程序提取 `test.bin` 中的数据，输出到 `test.txt` 文件中：

```bash
./bin_tool.exe test.bin test.txt
```

## 4. 实例

以背景中介绍的情况为例，调试时，在 Keil 的命令窗口内通过如下命令，将下载缓冲区内的内容输出到 `buffer.hex` 文件中：

```
SAVE tools\buffer.hex 0x20001000,0x2000D34F
```

`SAVE` 命令的具体使用方式见[[2]](https://www.keil.com/support/man/docs/uv4cl/uv4cl_cm_save.htm)。

使用 `hex_tool.exe` 提取 `buffer.hex` 中的内容，保存到 `buffer.txt` 中：

```bash
./hex_tool.exe buffer.hex buffer.txt
```

使用 `hex_tool.exe` 提取版本文件中的内容，保存到 `bin.txt` 中：

```bash
# 假设编译生成的文件为 test.hex 和 test.bin，二者包含的数据是相同的，提取其中一个的数据即可
./hex_tool.exe test.hex bin.txt
./bin_tool.exe test.bin bin.txt
```

之后，使用比较工具比较 `buffer.txt` 和 `bin.txt` 即可。

## 参考

[1] [GENERAL: Inter HEX File Format](https://developer.arm.com/documentation/ka003292/latest)

[2] [uVersion User's Guide: SAVE](https://www.keil.com/support/man/docs/uv4cl/uv4cl_cm_save.htm)