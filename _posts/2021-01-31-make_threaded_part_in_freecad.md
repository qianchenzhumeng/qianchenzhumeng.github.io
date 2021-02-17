---
layout: post
title:  "使用 FreeCAD 绘制螺纹"
date:   2021-01-31 20:12:00 +0800
categories: [3D Printing, FreeCAD]
tags: [3D Printing]
---

## 1. 背景

FreeCAD 版本：0.18

## 2. 绘制螺纹的几种方式

FreeCAD 官方 wiki 上，[Thread for Screw Tutorial[1]](https://wiki.freecadweb.org/Thread_for_Screw_Tutorial) 中介绍了绘制螺纹的 6 种方法：

- 0：从部件库里面获取符合 ISO 标准的紧固件
- 1：使用宏（已经不推荐了）
- 2：制作假螺纹（没有螺旋）
- 3：拿竖直方向的轮廓沿竖直方向的螺旋线扫掠
- 4：拿水平方向的轮廓沿竖直方向的螺旋线扫掠
- 5：从螺旋挤压的两个面之间放样

个人感觉如果是非标准件的话，选用方法 3 比较合适：拿竖直方向的轮廓沿竖直方向的螺旋线扫掠。

wiki 上的方法 3 步骤有点问题，比如在第 8 步中创建**增料圆柱体**时，在某些高度下，创建出来的圆柱体有问题，不显示，在后续的布尔运算中也看不到效果。另外，wiki 里面将螺纹末端的处理过程一笔带过，这块处理基本上没有参考意义。

## 3. 绘制螺纹

个别步骤中的操作跟 wiki 上的有点不同。螺纹末端的处理比较棘手，FreeCAD 的论坛里面有人提到了一种思路 [[2]](https://forum.freecadweb.org/viewtopic.php?t=54184&p=465782)，本文描述的螺纹末端处理就是照这个思路做的。

### (1) 绘制螺纹



- 在 **Part** 工作台中，点击 [<img src="https://wiki.freecadweb.org/images/4/4d/Part_Primitives.svg" alt="Part Primitives.svg" style="max-width: 1378px; height: auto; width: 21px; position: inherit; margin: 0 0 0 18px" />](https://wiki.freecadweb.org/File:Part_Primitives.svg)[创建参数化的几何图元（Part Primitives）](https://wiki.freecadweb.org/Part_Primitives)，创建一个螺旋体。**间距（Pitch）**为 3mm，高度为 21mm，半径为 10mm。

- 切换到 **Part Design** 工作台，点击 [<img src="https://wiki.freecadweb.org/images/1/10/PartDesign_Body.svg" alt="PartDesign Body.svg" style="max-width: 1378px; height: auto; width: 21px; position: inherit; margin: 0 0 0 18px" />](https://wiki.freecadweb.org/File:PartDesign_Body.svg) [创建新的可编辑实体并激活（PartDesign Body）](https://wiki.freecadweb.org/PartDesign_Body)。

- 点击 [<img src="https://wiki.freecadweb.org/images/6/6c/PartDesign_NewSketch.svg" alt="PartDesign NewSketch.svg" style="max-width: 1378px; height: auto; width: 21px; position: inherit; margin: 0 0 0 18px" />](https://wiki.freecadweb.org/File:PartDesign_NewSketch.svg) [创建一个新草绘（PartDesign New sketch）](https://wiki.freecadweb.org/PartDesign_NewSketch)，选择 XZ 平面。
- 按照需要的螺纹横截面形状绘制闭合的草图，通常为三角形。在这个示例中，将高度设置为 2.9mm，比螺旋体的间距 3mm 稍微小一点。沿着螺旋体扫掠时，要求刚才绘制的草图内部不能有任何交叉，只能是个轮廓。

![threads_helical_thread_profile_middle](/assets/img/2021-01-31-make_threaded_part_in_freecad.assets/threads_helical_thread_profile_middle.png)

![threads_helical_thread_path_middle](/assets/img/2021-01-31-make_threaded_part_in_freecad.assets/threads_helical_thread_path_middle.png)

- 选择草图，然后点击 [<img src="https://wiki.freecadweb.org/images/2/20/PartDesign_AdditivePipe.svg" alt="PartDesign AdditivePipe.svg" style="max-width: 1378px; height: auto; width: 21px; position: inherit; margin: 0 0 0 18px" />](https://wiki.freecadweb.org/File:PartDesign_AdditivePipe.svg) [沿路径或轮廓扫掠所选草图（PartDesign Additive pipe）](https://wiki.freecadweb.org/PartDesign_AdditivePipe)。点击**扫描路径（Path to sweep along）**下面的**对象（Object）**，选择刚才创建的螺旋体（如果没有出现在绘图区域，点击**组合浏览器**中的**模型**，选中螺旋体，按空格键切换螺旋体的可见性），将**截面方向**的**方向模式（Orientation mode）**修改为 `Frenet`，然后点击**OK**。

![threads_helical_thread_coil_middle](/assets/img/2021-01-31-make_threaded_part_in_freecad.assets/threads_helical_thread_coil_middle.png)

### (2) 处理螺纹末端

- 按照前面的方式新建一个螺旋体和一个螺纹横截面。螺旋体的间距 3mm，高度 1.5mm，半径 8.5mm，设置**位置**中的 x 坐标为 1.5mm，正好可以和最开始的螺旋体的底端接在一起。螺纹横截面位置、大小和刚才的一样。绘制好后拿新的轮廓延这个螺旋体扫掠，**截面方向**的**方向模式（Orientation mode）**仍然选 `Frenet`。

  ![threads_helical_thread_path_bottom](/assets/img/2021-01-31-make_threaded_part_in_freecad.assets/threads_helical_thread_path_bottom.png)

  ![threads_helical_thread_coil_bottom](/assets/img/2021-01-31-make_threaded_part_in_freecad.assets/threads_helical_thread_coil_bottom.png)

- 按照同样的方式，新建螺旋体和螺纹横截面，螺旋体的间距 3mm，高度 1.5mm，半径 8.5mm，设置**位置**中的 x 坐标为 1.5mm，z 的坐标为 21mm，正好可以和最开始的螺旋体的顶端接在一起。螺纹横截面大小和刚才的一样，位置在最顶端。

  ![threads_helical_thread_profile_top](/assets/img/2021-01-31-make_threaded_part_in_freecad.assets/threads_helical_thread_profile_top.png)

- 绘制好后拿新的轮廓延这个螺旋体扫掠，**截面方向**的**方向模式（Orientation mode）**仍然选 `Frenet`。

  ![threads_helical_thread_coil_top](/assets/img/2021-01-31-make_threaded_part_in_freecad.assets/threads_helical_thread_coil_top.png)

- 切换到 **Part** 工作台，创建一个圆柱体，半径为 10mm，高度 30mm，调整一下位置，将 z 设置为 -2mm。将圆柱另外复制两份。

- 在 **Part** 工作台中，将顶端扫掠出的部件和圆柱做布尔运算（差集，第一个形状为顶端扫掠出的部件，第二个为圆柱或其复制体）。

![threads_helical_thread_coil_remainder](/assets/img/2021-01-31-make_threaded_part_in_freecad.assets/threads_helical_thread_coil_remainder.png)

- 按照同样的方法处理底部扫掠出的部件。

- 最后将顶端剩余的部分、底部剩余的部分、中部扫掠出来的部分以及圆柱做布尔运算，求并集。

  ![threads_helical_thread_finished](/assets/img/2021-01-31-make_threaded_part_in_freecad.assets/threads_helical_thread_finished.png)

## 参考

[1] [https://wiki.freecadweb.org/Thread_for_Screw_Tutorial](https://wiki.freecadweb.org/Thread_for_Screw_Tutorial)

[2] [https://forum.freecadweb.org/viewtopic.php?t=54184&p=465782](https://forum.freecadweb.org/viewtopic.php?t=54184&p=465782)

