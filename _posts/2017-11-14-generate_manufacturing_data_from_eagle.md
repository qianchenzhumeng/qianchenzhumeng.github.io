---
layout: post
title:  "Eagle 输出 PCB 制造数据"
date:   2017-11-14 10:31:00 +0800
categories: [Embedded, PCB]
---

某宝上的电路板制造商的软件支持列表中，几乎是没有 Eagle 的。遇到这种情况，就不能像使用 Altium  Designer 那样直接把 pcb 文件发给制造商来制板了。不过，我们可以使用 Eagle 的 CAM 处理程序生成制造数据（钻孔数据以及 Gerber 光绘信息数据），把生成的数据交给电路板制造商就可以制板了。

## 1. 制造数据清单

|      | 文件   | 选中的层                        | 描述             |
| :--: | ------ | ------------------------------- | ---------------- |
|  1   | %N.drd | 44  Drills                      | 电镀孔           |
|  2   | %N.hol | 45  Holes                       | 非电镀孔         |
|  3   | %N.cmp | 1 Top,  17  Pads,  18 Vias      | 焊接层（顶层）   |
|  4   | %N.sol | 16  Bottom,  17  Pads,  18 Vias | 焊接层（底层）   |
|  5   | %N.plc | 21  tPlace,  25  tNames         | 顶层丝印层       |
|  6   | %N.pls | 22  bPlace,  26  bNames         | 底层丝印层       |
|  7   | %N.stc | 29  tStop                       | 顶层阻焊层       |
|  8   | %N.sts | 30  bStop                       | 底层阻焊层       |
|  9   | %N.crc | 31  tCream                      | 顶层焊膏层       |
|  10  | %N.crs | 32  bCream                      | 底层焊膏层       |
|  11  | %N.dim | 20  Dimension                   | 非电镀铣加工轮廓 |

## 2. 使用 CAM 处理器生成制造数据

Eagle 自带了一些生成制造数据的 CAM 处理器文件（位于程序安装目录内的 cam 目录下），能够满足一些基础需求，实际使用过程中，需要以自带的 CAM 处理器文件为基础进行修改，以生成合适的制造数据。

下面以 [Arduino UNO R3](https://www.arduino.cc/en/uploads/Main/arduino_Uno_Rev3-02-TH.zip) 的设计文件为例，描述生成制造数据的步骤。

- 生成电镀孔和非电镀孔的钻孔数据
- 生成光绘信息文件

![image-20211007101239576](/assets/img/2017-11-14-generate_gerber_files_from_eagle.assets/image-20211007101239576.png)

### (1) 生成过孔数据

PCB 预览图中，走线过孔、接插件的固定插孔都是电镀孔，四周的四个白色圆孔是非电镀孔。

Eagle 自带的 CAM 处理器文件中有生成钻孔数据的作业文件（excellon.cam）。从下图可以看到，该作业文件选中了 44 Drills 层和 45 Holes 层，即电镀孔和非电镀孔数据生成在一个文件里。

<img src="/assets/img/2017-11-14-generate_gerber_files_from_eagle.assets/image-20211007102418406.png" alt="image-20211007102418406" style="width:40%;" />

点击“处理作业”后，设计文件所在的目录内会出现 `arduino_Uno_Rev3-02-TH.drd` 文件，该文件包含的数据就是所有的电镀孔和非电镀孔信息。

|      | 文件   | 选中的层              | 描述                     |
| :--: | ------ | --------------------- | ------------------------ |
|  1   | %N.drd | 44  Drills, 45  Holes | 钻孔（电镀孔、非电镀孔） |

这个与本文一开始给出的数据文件清单有点区别，本文一开始给出的数据文件清单，电镀孔和非电镀孔分属两个数据文件。这一点和制造商有关系，如果制造商需要区分电镀孔和非电镀孔，就需要将电镀孔和非电镀孔的数据输出在两个文件中。

只需要在默认的 excellon.cam 作业文件上进行简单的修改即可完成区分电镀孔和非电镀孔的工作。

- 为了避免修改默认的作业文件，从 cam 目录中复制一份 excellon.cam 文件到单板设计文件所在的目录中。
- 在 cam 处理器中打开该作业文件。
- 将默认的选项卡名称修改为“生成电镀孔数据”，仅选中 44 Drills 层。

<img src="/assets/img/2017-11-14-generate_gerber_files_from_eagle.assets/image-20211007113133798.png" alt="image-20211007113133798" style="width:60%;" />

- 点击“添加”按钮，新建选项卡，将其名称改为“生成非电镀孔数据”，将文件后缀改为 hol，仅选中 45 Holes 层。 

<img src="/assets/img/2017-11-14-generate_gerber_files_from_eagle.assets/image-20211007113422048.png" alt="image-20211007113422048" style="width:60%;" />

- 点击“处理作业”按钮，生成钻孔数据文件。

|      | 文件                       | 选中的层    | 描述     |
| :--: | -------------------------- | ----------- | -------- |
|  1   | arduino_Uno_Rev3-02-TH.drd | 44  Drills, | 电镀孔   |
|  2   | arduino_Uno_Rev3-02-TH.hol | 45  Holes   | 非电镀孔 |

### (2) 生成光绘信息文件

生成光绘信息文件之前，可能还需要设置过孔盖油或过孔开窗，请参考另一篇博客《Eagle 设置过孔盖油/开窗》。

Eagle 自带的 CAM 处理器文件中有生成光绘信息的作业文件（gerb274x.cam），默认会输出顶层焊接层、底层焊接层、顶层丝印层、顶层阻焊层、底层阻焊层数据文件。

|      | 文件   | 选中的层                              | 描述           |
| :--: | ------ | ------------------------------------- | -------------- |
|  1   | %N.cmp | 1 Top,  17  Pads,  18 Vias            | 焊接层（顶层） |
|  3   | %N.sol | 16  Bottom,  17  Pads,  18 Vias       | 焊接层（底层） |
|  4   | %N.plc | 20 Dimension, 21  tPlace,  25  tNames | 顶层丝印层     |
|  5   | %N.stc | 29  tStop                             | 顶层阻焊层     |
|  6   | %N.sts | 30  bStop                             | 底层阻焊层     |

可以看到，生成的文件要比本文一开始给的数据文件清单内的文件少。原因是上述的作业文件只考虑了顶层有丝印的情况，对于底层还有丝印的板子，还需要额外的动作来生成底层丝印层。还有就是，非电镀铣加工轮廓（电路板外框）放在了顶层丝印层文件中。

和生成钻孔数据文件时区分电镀孔和非电镀孔类似，也可以通过对 gerb274x.cam 进行简单修改来按照本文一开始给的清单生成光绘信息文件。

- 为了避免修改默认的作业文件，从 cam 目录中复制一份 gerb274x.cam 文件到单板设计文件所在的目录中。
- 在 cam 处理器中打开该作业文件。
- 点击“添加”按钮新建选项卡，将名称改为“底层丝印层”，将文件后缀改为“pls”，选中 22 bPlace 层和 26 bNames 层。

<img src="/assets/img/2017-11-14-generate_gerber_files_from_eagle.assets/image-20211007115157967.png" alt="image-20211007115157967" style="width:60%;" />

- 点击“Silk screen CMP”选项卡，取消对 20 Dimension 层的选择。
- 点击“添加”按钮，新建选项卡，将名称改为“电路板外框”，将文件后缀改为“dim”，仅选中 20 Dimension 层。

<img src="/assets/img/2017-11-14-generate_gerber_files_from_eagle.assets/image-20211007115508897.png" alt="image-20211007115508897" style="width:60%;" />

- 点击“处理作业”，生成光绘信息文件。

|      | 文件   | 选中的层                        | 描述             |
| :--: | ------ | ------------------------------- | ---------------- |
|  1   | %N.cmp | 1 Top,  17  Pads,  18 Vias      | 焊接层（顶层）   |
|  2   | %N.sol | 16  Bottom,  17  Pads,  18 Vias | 焊接层（底层）   |
|  3   | %N.plc | 21  tPlace,  25  tNames         | 顶层丝印层       |
|  4   | %N.pls | 22  bPlace,  26  bNames         | 底层丝印层       |
|  5   | %N.stc | 29  tStop                       | 顶层阻焊层       |
|  6   | %N.sts | 30  bStop                       | 底层阻焊层       |
|  7   | %N.dim | 20  Dimension                   | 非电镀铣加工轮廓 |

顶层焊膏层、底层焊膏层对应的数据文件可以用于顶层、底层有贴片元件时生成钢网，生成电路板制造数据时可以不需要。

## 3. 预览制造数据

可以使用 ViewMata 免费版预览 Gerber 数据。

- 选中第一层，选择电镀孔文件（arduino_Uno_Rev3-02-TH.drd）：File->Import->Dill&Rout..

<img src="/assets/img/2017-11-14-generate_gerber_files_from_eagle.assets/image-20211007121450458.png" alt="image-20211007121450458" style="width:60%;" />

- 点击“Options”按钮，修改配置，点击“确定”按钮，然后点击“Import”按钮导入。

<img src="/assets/img/2017-11-14-generate_gerber_files_from_eagle.assets/image-20211007121645224.png" alt="image-20211007121645224" style="width:40%;" />

- 选中第二层，按同样的方法导入非电镀孔数据文件“arduino_Uno_Rev3-02-TH.hol”。
- 选中第三层，导入顶层焊接层数据文件（arduino_Uno_Rev3-02-TH.cmp）：File->Import->Gerber...
- 选中第四层，导入底层焊接层文件（arduino_Uno_Rev3-02-TH.sol）：File->Import->Gerber...
- 选中第五层，导入顶层丝印层文件（arduino_Uno_Rev3-02-TH.plc）：File->Import->Gerber...
- 选中第六层，导入底层丝印层文件（arduino_Uno_Rev3-02-TH.pls）：File->Import->Gerber...
- 选中第七层，导入顶层阻焊层文件（arduino_Uno_Rev3-02-TH.stc）：File->Import->Gerber...
- 选中第八层，导入底层阻焊层文件（arduino_Uno_Rev3-02-TH.sts）：File->Import->Gerber...
- 选中第九层，导入非电镀铣加工轮廓文件（arduino_Uno_Rev3-02-TH.dim）：File->Import->Gerber...

> 预览的时候可以发现，顶层焊接层中出现了 PCB 设计文件中的声明信息，其中，免责声明是作为器件的形式放到原理图和PCB图中的，另一部分声明是放在 1 Top 层的文字。应该可以不用处理，如果要处理的话，在原理图中删去免责声明，另外的一部分声明可以删除或者放在其他不会被 CAM 处理器输出的层（例如，48 Document 层）。

> 顶层丝印层和底层丝印层的 PCB 框外也会出现部分文字，原因是这些文字放在 25 tNames 层，如果需要处理的话，在 PCB 文件中删除或者放到其他不会被 CAM 处理器输出的层即可。

预览效果如下：

<img src="/assets/img/2017-11-14-generate_gerber_files_from_eagle.assets/image-20211007125246240.png" alt="image-20211007125246240" style="width:60%;" />

## 4. 打样

确认没有问题后，就可以把生成的过孔数据和光绘信息文件打包发给商家进行打样了。
