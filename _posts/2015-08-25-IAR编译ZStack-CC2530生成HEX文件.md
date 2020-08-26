---
layout: post
title:  "IAR 编译 ZStack-CC2530 生成 HEX 文件"
date:   2015-08-25 00:11:26 +0800
categories: [Embedded, MCU]
---

IAR 编译 ZStack-CC2530 为可下载运行的 HEX 文件的正确配置：

1. 正确配置输出文件格式：菜单选择Project－Options－Linker－Output－Format，选择Other。右边的Output下拉框选intel-extended，Format variant下拉框选None，Module-local下拉框选Include all

2. 还是在菜单Project－Options－Linker－Output标签中，勾上Override default选项，把编辑框中的文件名的后缀改为hex

以上两步都是大多数人熟知的，下面这一步是针对大型程序编译下载所必须的，也是大部分写zstack教程的人所没有提到的。

3. 找到f8w2530.xcl文件，并打开。（这个文件在"Projects/zstack/Tools/CC2530DB/"目录下，也可以通过IAR编译环境的左侧Workspace窗口点开Tools文件夹看到）在f8w2530.xcl文件中找到两行被注释掉的语句：

```
//-M(CODE)[(_CODEBANK_START+_FIRST_BANK_ADDR)-(_CODEBANK_END+_FIRST_BANK_ADDR)]*/
//_NR_OF_BANKS+_FIRST_BANK_ADDR=0x8000
```

把这两行前面的"//"去掉，保存，重新编译，OK！

> （注：去掉这两行的"//"后在编译输出成hex格式时没有问题，但在debug模式下编译会提示警告：Warning[w69]: Address translation (-M, -b# or -b@) has no effect on the output format 'debug'. The output file will be generated but noaddress translation will be performed. 不过并不会影响debug调试的使用。也许正是为了屏蔽掉此条警告，所以TI在发布Zstack时选择了默认为debug模式才注释掉了这两行指令，但在编译hex时却又不提示任何警告和错误，真是害人不浅～～）