---
layout: post
title:  "Modern Robotics: Mechanics, Planning, and Control"
date:   2021-11-14 22:25:00 +0800
categories: [Other]
tags: [Robot]
excerpt: 记录机器人课程学习过程。
---

之前学习了 Coursera 上的 [機器人學一 (Robotics (1))](https://www.coursera.org/learn/robotics1)，对机械臂的运动学分析以及路径规划有了大致了解，课程后期的测验题涉及了大量的三角函数运算，手算几乎不太可能，由于缺乏编程求解方面的详细指导，后面的题目也没有做出来。

看了一下 [Modern Robotics: Mechanics, Planning, and Control 专项课程](https://www.coursera.org/specializations/modernrobotics) 的简介，有编程方面的指导，也有模拟软件使用方面的教学，学起来应该透彻一些，打算花半年到一年的时间学完该专项课程中的 6 门课。

## 1. 课程 1

学习如何使用 CoppeliaSim 模拟器：[Getting Started with the CoppeliaSim Simulator](http://hades.mech.northwestern.edu/index.php/Getting_Started_with_the_CoppeliaSim_Simulator)，需要从 [CoppeliaSim Introduction](http://hades.mech.northwestern.edu/index.php/CoppeliaSim_Introduction) 下载场景文件：[V-REP_scenes-Nov2019.zip](http://hades.mech.northwestern.edu/images/0/07/V-REP_scenes-Nov2019.zip)

C-space：所有配置的空间

Degrees of freedom(dof)：C-space 的维度

### (1) 刚体自由度的计算

对于 3 维空间的刚体，取第一个点 A，有 3 个坐标，0 个限制条件，则 A 点有 3 个自由度；取第二个点 B，有 3 个坐标，1 个限制条件（到 A 点的距离），则 B 有 3-1=2 个自由度；取第三个点 C，有 3 个坐标，2 个限制条件（到 A 点的距离和到 B 点的距离），则 C 有 3-2=1 个自由度；剩下的所有其他点，每个都有 3 个坐标，3 个限制条件（到 A、B、C 点的距离），则这些点有 3-3=0 个自由度。故 3 维空间中的刚体总共有 3+2+1=6 个自由度，其中，3 个是线性的，剩下的 3 个是角度（3 个移动，3 个转动）。

|    点    | 坐标 |      | 限制条件 |      | 自由度 |
| :------: | :--: | :--: | :------: | :--: | :----: |
|    A     |  3   |  -   |    0     |  =   |   3    |
|    B     |  3   |  -   |    1     |  =   |   2    |
|    C     |  3   |  -   |    2     |  =   |   1    |
| 剩下的点 |  3   |  -   |    3     |  =   |   0    |
|    共    |      |      |          |      |   6    |

对于 n 维空间中的刚体：

|    点    | 坐标 |      | 限制条件 |      |  自由度  |
| :------: | :--: | :--: | :------: | :--: | :------: |
|    1     |  n   |  -   |    0     |  =   |    n     |
|    2     |  n   |  -   |    1     |  =   |   n-1    |
|    3     |  n   |  -   |    2     |  =   |   n-2    |
|          |      |      |          |      |          |
|    n     |  n   |  -   |  (n-1)   |  =   |    1     |
| 剩下的点 |  n   |  -   |    n     |  =   |    0     |
|    共    |      |      |          |      | n(n+1)/2 |

其中，n 个移动，n(n+1)/2-n=n(n-1)/2 个转动。

### (2) 机器人自由度的计算

Grubler 公式：
$$
dof = m(N-1-J)+\sum_{i=1}^Jf_i
$$
其中，m 为单个刚体的自由度，N 为连杆的数量（包括地），J 为关节的数量，最后的求和项为所有关节的自由度的和。

适用场景：关节提供的限制条件是独立的。
