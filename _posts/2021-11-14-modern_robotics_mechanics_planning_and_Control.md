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

### 1.1 第一周

#### (1) 刚体自由度的计算

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

#### (2) 机器人自由度的计算

Grubler 公式：


$$
dof = m(N-1-J)+\sum_{i=1}^Jf_i
$$


其中，m 为单个刚体的自由度，N 为连杆的数量（包括基座），J 为关节的数量，最后的求和项为所有关节的自由度的和。

适用场景：关节提供的限制条件是独立的。

#### (3) 位形空间拓扑

#### (4) 位形空间表示

#### (5) 位形及速度约束

#### (6) 任务空间及工作空间

### 1.2 第二周

#### (1) 刚体运动介绍

#### (2) 旋转矩阵

#### (3) 角速度

#### (4) 转动的指数坐标

#### (5) 测验

**1.** 已知 $\hat{x}_a$ 和 $\hat{y}_a$，可以通过图示法确定 $\hat{z}_a$，进而得到 $R_{sa}$ ：

$$
R_{sa}=
\left[
\begin{array}{1}
0 & 1 & 0\\
0 & 0 & 1\\
1 & 0 & 0
\end{array}
\right]
$$

**2.**  已知 $\hat{x}_b$ 和 $\hat{y}_b$，可以通过图示法确定 $\hat{z}_b$，进而得到 $R_{sb}$：

$$
R_{sb}=
\left[
\begin{array}{1}
1 & 0 & 0\\
0 & 0 & 1\\
0 & -1 & 0
\end{array}
\right]
$$
进而：
$$
R^{-1}_{sb}=R^{T}_{sb}=
\left[
\begin{array}{1}
1 & 0 & 0\\
0 & 0 & 1\\
0 & -1 & 0
\end{array}
\right]^T
=
\left[
\begin{array}{1}
1 & 0 & 0\\
0 & 0 & -1\\
0 & 1 & 0
\end{array}
\right]
$$

**3.** 利用下标消减的原则以及旋转矩阵的特性，已知 $R_{sa}$ 和 $R_{sb}$ 可得：

$$
R_{ab}=R_{as}R_{sb}=R_{sa}^{-1}R_{sb}=R_{sa}^TR_{sb}=
\left[
\begin{array}{1}
0 & 1 & 0\\
0 & 0 & 1\\
1 & 0 & 0
\end{array}
\right]^T
\left[
\begin{array}{1}
1 & 0 & 0\\
0 & 0 & 1\\
0 & -1 & 0
\end{array}
\right]
=
\left[
\begin{array}{1}
0 & -1 & 0\\
0 & 0 & -1\\
0 & 1 & 0
\end{array}
\right]
$$

**4.** 已知 $R=R_{sb}$，可得：

$$
R_1=R_{sa}R=R_{sa}R_{sb}=
\left[
\begin{array}{1}
0 & 1 & 0\\
0 & 0 & 1\\
1 & 0 & 0
\end{array}
\right]
\left[
\begin{array}{1}
1 & 0 & 0\\
0 & 0 & 1\\
0 & -1 & 0
\end{array}
\right]
=
\left[
\begin{array}{1}
0 & 0 & 1\\
0 & -1 & 0\\
1 & 0 & 0
\end{array}
\right]
$$

使用图示法，将 $R_1$ 和 {a} {s} 对比，可知 $R_1$ 是 $R_{sa}$ 绕 $\hat{x}_a$ -90° 后得到的新旋转矩阵。

**5.** 已知 $p_b=(1,2,3)^T$，利用 $R_{sb}$ 将 $p_b$ 转换到 {s}：

$$
p_s=R_{sb}p_b=
\left[
\begin{array}{1}
1 & 0 & 0\\
0 & 0 & 1\\
0 & -1 & 0
\end{array}
\right]
\left[
\begin{array}{1}
1 \\
2 \\
3
\end{array}
\right]
=
\left[
\begin{array}{1}
1 \\
3 \\
-2
\end{array}
\right]
$$

**6.** 已知 $R_{sb}$ 与 $p_s=(1,2,3)^T$，根据旋转矩阵的性质以及下标消减的原则，可得：

$$
q=R_{sb}^Tp_s=R_{sb}^{-1}p_s=R_{bs}p_s=p_b
$$

即，q 是 p 在 {b} 中的表示。

**7.** 已知 $\omega_s=(3,2,1)^T$，由下标消除的原则以及旋转矩阵的性质可得：

$$
\omega_a=R_{as}\omega_s=R_{sa}^{-1}\omega_s=R_{sa}^T\omega_s
=
\left[
\begin{array}{1}
0 & 1 & 0\\
0 & 0 & 1\\
1 & 0 & 0
\end{array}
\right]^T
\left[
\begin{array}{1}
3 \\
2 \\
1
\end{array}
\right]
=
\left[
\begin{array}{1}
0 & 0 & 1\\
1 & 0 & 0\\
0 & 1 & 0
\end{array}
\right]
\left[
\begin{array}{1}
3 \\
2 \\
1
\end{array}
\right]
=
\left[
\begin{array}{1}
1 \\
3 \\
2
\end{array}
\right]
$$

```
[1,3,2]
```

**8.** 根据教材 $p_{53}$ 的内容，已知 $R_{sa}$，可得：

$$
\theta=cons^{-1}(\frac{1}{2}(trR-1))=cos^{-1}(-\frac{1}{2})=\frac{2\pi}{3}\approx2.09
$$

**9.** 已知 $\hat{\omega}\theta=(1,2,0)^T$，单位化后可得：

$$
\hat{\omega}=
\left[
\begin{array}{1}
\frac{1}{\sqrt{5}} \\
\frac{2}{\sqrt{5}} \\
0
\end{array}
\right]=
\left[
\begin{array}{1}
0.447 \\
0.894 \\
0
\end{array}
\right] \\
\implies
\theta=\sqrt{5}\approx2.236
$$

进而，可得 $\hat{\omega}$ 对应的反对称矩阵：

$$
[\hat{\omega}]=
\left[
\begin{array}{1}
0 & -\hat{\omega}_3 & \hat{\omega}_2 \\
\hat{\omega}_3 & 0 & -\hat{\omega}_1 \\
-\hat{\omega}_2 & \hat{\omega}_1 & 0
\end{array}
\right]
=
\left[
\begin{array}{1}
0 & 0 & 0.894 \\
0 & 0 & -0.447 \\
-0.894 & 0.447 & 0 \\
\end{array}
\right]
$$

由罗德里格斯公式 $Rot(\hat{\omega},\theta)=e^{[\hat{\omega}]\theta}=I+sin\theta[\hat{\omega}]+(1-cos\theta)[\hat{\omega}]^2$ 可得：

$$
\begin{aligned}
Rot(\hat{\omega},\theta) &= I+0.787
\left[
\begin{array}{1}
0 & 0 & 0.894 \\
0 & 0 & -0.447 \\
-0.894 & 0.447 & 0 \\
\end{array}
\right]
+
1.617
\left[
\begin{array}{1}
0 & 0 & 0.894 \\
0 & 0 & -0.447 \\
-0.894 & 0.447 & 0 \\
\end{array}
\right]^2 \\
&=I+
\left[
\begin{array}{1}
0 & 0 & 0.704 \\
0 & 0 & -0.352 \\
-0.704 & 0.352 & 0 \\
\end{array}
\right]
+
1.617
\left[
\begin{array}{1}
-0.799 & 0.400 & 0 \\
0.400 & -0.200 & 0 \\
0 & 0 & -1.00 \\
\end{array}
\right] \\
&=
\left[
\begin{array}{1}
1 & 0 & 0.704 \\
0 & 1 & -0.352 \\
-0.704 & 0.352 & 1 \\
\end{array}
\right]
+
\left[
\begin{array}{1}
-1.292 & 0.647 & 0 \\
0.647 & -0.323 & 0 \\
0 & 0 & -1.617 \\
\end{array}
\right] \\
&=
\left[
\begin{array}{1}
-0.292 & 0.647 & 0.704 \\
0.647 & 0.677 & -0.352 \\
-0.704 & 0.352 & 0.617 \\
\end{array}
\right]
\end{aligned}
$$

```
[[-0.292,0.647,0.704],[0.647,0.677,-0.352],[-0.704,0.352,-0.617]]
```

如果使用软件计算：

```python
import modern_robotics as mr
import numpy as np

omega_theta=np.array([1,2,0])
theta = mr.AxisAng3(omega_theta)[1]
print (theta)
so3_mat=mr.VecToso3(omega_theta)
print (so3_mat)
r = mr.MatrixExp3(so3_mat)
print (r)
```

**10.** 已知 $\omega=(1,2,0.5)^T$，可得：

$$
[\omega]=
\left[
\begin{array}{1}
0 & -\omega_3 & \omega_2 \\
\omega_3 & 0 & -\omega_1 \\
-\omega_2 & \omega_1 & 0
\end{array}
\right]
=
\left[
\begin{array}{1}
0 & -0.5 & 2\\
0.5 & 0 & -1\\
-2 & 1 & 0
\end{array}
\right]
$$

```
[[0,-0.5,2],[0.5,0,-1],[-2,1,0]]
```

如果使用软件计算：

```python
import modern_robotics as mr
import numpy as np

omg = np.array([1, 2, 0.5])
so3=mr.VecToso3(omg)
print (so3)
```

**11.** 已知矩阵对数 $[\hat{\omega}]\theta$，使用 `MatrixExp3` 计算 $R \in SO(3)$：

```python
import modern_robotics as mr
import numpy as np

omega_hat_so3_theta = np.array([
    [0, 0.5, -1],
    [-0.5, 0, 2],
    [1, -2, 0]
])
r = mr.MatrixExp3(omega_hat_so3_theta)
print (r)
```

结果：

```
[[0.60482045,0.796274,-0.01182979],[0.46830057,-0.34361048,0.81401868],[0.64411707,-0.49787504,-0.58071821]]
```

**12.** 已知矩阵 R，使用 `MatrixLog3` 计算矩阵对数 $[\hat{\omega}]\theta \in so(3)$：

```python
import modern_robotics as mr
import numpy as np

R=np.array([
    [0, 0, 1],
    [-1, 0, 0],
    [0, -1, 0]
])
matrix_log=mr.MatrixLog3(R)
print (matrix_log)
```

结果：

```
[[ 0,1.20919958,1.20919958],[-1.20919958,0,1.20919958],[-1.20919958,-1.20919958,0]]
```
