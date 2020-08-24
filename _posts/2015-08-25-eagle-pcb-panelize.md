---
layout: post
title:  "EAGLE PCB 拼板"
date:   2015-08-25 00:11:26 +0000
categories: [Embedded, PCB]
---


## 1. 操作步骤

(1) 在 eagle 中新建一个 PCB 文件并保存，作为拼板文件。

(2) 只打开需要拼板的 brd 文件，保持原理图为关闭状态。

(3) 打开用户语言程序 panelize.ulp，在弹出的对话框中点击 `Execute`，如果是单面板的话，图层中会出现第125层（`_tName`层），如果是双面板，除出现第125层外还会出现第126层（`_bName` 层）。

运行 `panelize.ulp` 程序之前：

![](/assets/img/2015-08-25-eagle-pcb-panelize.assets/1.png)

运行 `panelize.ulp` 程序之后：

![](/assets/img/2015-08-25-eagle-pcb-panelize.assets/2.png)

元件的名称被复制到了 `_tName` 层。

(4) 运行 `Copy` 命令，然后运行 `Group` 命令，框选整个 PCB 图，右击，选择 `Copy：组合`。

(5) 打开新建的 PCB 文件，运行多次 `Paste` 命令，将 PCB 图逐个粘贴到该文件中。

![](/assets/img/2015-08-25-eagle-pcb-panelize.assets/3.png)

执行粘贴操作后，位于 `tName` 层的编号会自动追加，即各板上相同元件的名称并不相同。但是 `_tName` 层上的编号不会变，每块板上的编号都相同。

(6) 输出制造数据时将 `tName` 层和 `bName` 层分别替换为 `_tName` 层和 `_bName` 层。

## 2. 预览制造数据

在 ViewMate 中预览 Gerber 数据：

![](/assets/img/2015-08-25-eagle-pcb-panelize.assets/4.png)
