---
layout: post
title:  "Eagle 输出 Gerber 数据"
date:   2017-11-14 10:31:00 +0800
categories: [Embedded, PCB]
---

某宝上的电路板制造商的软件支持列表中，几乎是没有Eagle的。遇到这种情况，就不能像使用 Altium  Designer 那样直接把 pcb 文件发给制造商来制板了。不过，我们可以使用 Eagle 的 CAM 处理程序生成 Gerber 数据，把生成的gerber数据交给电路板制造商就可以制板了。

制造数据清单

| 文件   | 选中的层                        | 描述             |
| ------ | ------------------------------- | ---------------- |
| %N.drd | 44  Drills,  45  Holes          | 电镀孔           |
| %N.cmp | 1 Top,  17  Pads,  18 Vias      | 焊接层（顶层）   |
| %N.sol | 16  Bottom,  17  Pads,  18 Vias | 焊接层（底层）   |
| %N.plc | 21  tPlace,  25  tNames         | 顶层丝印层       |
| %N.pls | 22  bPlace,  26  bNames         | 底层丝印层       |
| %N.stc | 29  tStop                       | 顶层阻焊层       |
| %N.sts | 30  bStop                       | 底层阻焊层       |
| %N.crc | 31  tCream                      | 顶层焊膏层       |
| %N.crs | 32  bCream                      | 底层焊膏层       |
| %N.hol | 45  Holes                       | 非电镀孔         |
| %N.dim | 20  Dimension                   | 非电镀铣加工轮廓 |

可以使用 Viewmata 免费版预览 Gerber 数据。