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

### 1.2 第三周

#### (1) 位形空间拓扑

#### (2) 位形空间表示

#### (3) 位形及速度约束

#### (4) 任务空间及工作空间

### 1.3 第三周

#### (1) 刚体运动介绍

#### (2) 旋转矩阵

特殊正交群：$$3 \times 3$$ 旋转矩阵组成的集合称为 **特殊正交群（special orthogonal group）SO(3)**，满足：

$$
R^TR=I \\
det R=1
$$

旋转矩阵的特性：

$$
可逆性：R^{-1}=R^T \in SO(3) \\
封闭性：R_1R_2 \in SO(3) \\
结合律：(R_1R_2)R_3=R_1(R_2R_3) \\
不满足交换律：R_1R_2 \ne R_2R_1 \\
x \in \mathbb{R}^3,\Vert Rx \Vert=\Vert x \Vert
$$

旋转矩阵的用途：

- 表示姿态
- 变换参考坐标系
- 旋转某一向量或坐标

#### (3) 角速度

反对称矩阵：给定向量 $$x=[x_1,x_2,x_3]^T \in \mathbb{R}$$，定义其反对称矩阵 $$[x]$$ 为：

$$
[x]=
\begin{bmatrix}
0 & -x_3 & x_2 \\
x_3 & 0 & -x_1 \\
-x_2 & x_1 & 0
\end{bmatrix}
$$

反对称矩阵满足：

$$
[x]=-[x]^T
$$

所有 $$3 \times 3$$ 反对称矩阵的集合称为 $$so(3)$$。

#### (4) 刚体转动的指数坐标

旋转矩阵 $$R$$ 的指数坐标 $$\hat{\omega}\theta \in \mathbb{R}^3$$，其中，$$\hat{\omega} \in \mathbb{R}^3$$ 为单位转轴，$$\theta \in \mathbb{R}$$ 为绕转该轴线的转角。

#### (5) 测验

**1.** 已知 $$\hat{x}_a$$ 和 $$\hat{y}_a$$，可以通过图示法确定 $$\hat{z}_a$$，进而得到 $$R_{sa}$$ ：

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

**2.**  已知 $$\hat{x}_b$$ 和 $$\hat{y}_b$$，可以通过图示法确定 $$\hat{z}_b$$，进而得到 $$R_{sb}$$：

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

**3.** 利用下标消减的原则以及旋转矩阵的特性，已知 $$R_{sa}$$ 和 $$R_{sb}$$ 可得：

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

**4.** 已知 $$R=R_{sb}$$，可得：

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

使用图示法，将 $$R_1$$ 和 {a} {s} 对比，可知 $$R_1$$ 是 $$R_{sa}$$ 绕 $$\hat{x}_a$$ -90° 后得到的新旋转矩阵。

**5.** 已知 $$p_b=(1,2,3)^T$$，利用 $$R_{sb}$$ 将 $$p_b$$ 转换到 {s}：

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

**6.** 已知 $$R_{sb}$$ 与 $$p_s=(1,2,3)^T$$，根据旋转矩阵的性质以及下标消减的原则，可得：

$$
q=R_{sb}^Tp_s=R_{sb}^{-1}p_s=R_{bs}p_s=p_b
$$

即，q 是 p 在 {b} 中的表示。

**7.** 已知 $$\omega_s=(3,2,1)^T$$，由下标消除的原则以及旋转矩阵的性质可得：

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

**8.** 根据教材 $$p_{53}$$ 的内容，已知 $$R_{sa}$$，可得：

$$
\theta=cons^{-1}(\frac{1}{2}(trR-1))=cos^{-1}(-\frac{1}{2})=\frac{2\pi}{3}\approx2.09
$$

**9.** 已知 $$\hat{\omega}\theta=(1,2,0)^T$$，单位化后可得：

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

进而，可得 $$\hat{\omega}$$ 对应的反对称矩阵：

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

由罗德里格斯公式 $$Rot(\hat{\omega},\theta)=e^{[\hat{\omega}]\theta}=I+sin\theta[\hat{\omega}]+(1-cos\theta)[\hat{\omega}]^2$$ 可得：

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

**10.** 已知 $$\omega=(1,2,0.5)^T$$，可得：

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

**11.** 已知矩阵对数 $$[\hat{\omega}]\theta$$，使用 `MatrixExp3` 计算 $$R \in SO(3)$$：

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

**12.** 已知矩阵 R，使用 `MatrixLog3` 计算矩阵对数 $$[\hat{\omega}]\theta \in so(3)$$：

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

### 1.4 第四周

#### (1) 齐次变换矩阵

**特殊欧式群（special Euclidean group）SE(3)** 亦称 **刚体运动（rigid-body motion）群** 或 **齐次变换矩阵（homogeneous transformation matrice）群**，是所有 $$4 \times 4$$ 实矩阵 T 的集合：

$$
T=
\left[
\begin{array}{1}
R & p \\
0 & 1
\end{array}
\right]
=
\left[
\begin{array}{1}
r_{11} & r_{12} & r_{13} & p_{1} \\
r_{21} & r_{22} & r_{23} & p_{2} \\
r_{31} & r_{32} & r_{33} & p_{3} \\
0 & 0 & 0 & 1
\end{array}
\right] \in SE(3)
$$

变换矩阵的特性：

$$
可逆性：T^{-1}=
\left[
\begin{array}{1}
R & p \\
0 & 1
\end{array}
\right]^{-1}
=
\left[
\begin{array}{1}
R^T & -R^Tp \\
0 & 1
\end{array}
\right] \in SE(3) \\
封闭性：T_1T_2 \in SE(3) \\
结合律：(T_1T_2)T_3=T_1(T_2T_3) \\
不满足交换律：T_1T_2 \ne T_2T_1
$$

齐次变换矩阵的用途：

- 表示刚体的位形（位置和姿态）
- 变换参考坐标系
- 表示向量或左边的位移

#### (2) 运动旋量

旋转轴：

$$
S=
\begin{bmatrix}
S_\omega \\
S_v
\end{bmatrix}
=
\left[
\begin{array}{1}
\dot{\theta}=1 时的角速度 \\
\dot{\theta}=1 时原点的线速度
\end{array}
\right]
$$

运动旋量：

$$
V=
\begin{bmatrix}
\omega \\
v
\end{bmatrix}
=S\dot{\theta}
$$

变换矩阵 T 的伴随变换矩阵（adjoint representation）$$[Ad_T]$$：

$$
[Ad_T]=
\left[
\begin{array}{1}
R & 0 \\
[p]R & R
\end{array}
\right] \in \mathbb{R}^{6 \times 6}
$$

对于任意两个坐标系 {a} 与 {b}，运动旋量之间的关系为：

$$
\mathcal{V_a}=[Ad_{T_{ab}}]\mathcal{V_b}
$$

两个不同坐标系 {a} 与 {b} 中所描述的螺旋轴之间的映射关系：

$$
\mathcal{S_a}=[Ad_{T_{ab}}]\mathcal{S_b}
$$


物体运动旋量（$$\mathcal{S}$$ 在 {b} 中表示）：
$$
\mathcal{V_b}=(\omega_b,v_b)=\mathcal{S}\dot{\theta}
$$

物体运动旋量的矩阵表示：

$$
[\mathcal{V_b}]=T^{-1}\dot{T}
=
\left[
\begin{array}{1}
[\mathcal{\omega_b}] & \mathcal{v_b} \\
0 & 0
\end{array}
\right] \in \mathcal{se}(3)
$$

空间运动旋量（$$\mathcal{S}$$ 在 {s} 中表示）：

$$
\mathcal{V_s}=(\omega_s,v_s)=\mathcal{S}\dot{\theta}
$$

空间运动旋量的矩阵表示：

$$
[\mathcal{V_s}]=\dot{T}T^{-1}=
\begin{bmatrix}
[\mathcal{\omega_s}] & \mathcal{v_s} \\
0 & 0
\end{bmatrix} \in \mathcal{se}(3)
$$

所有左上角为 $$3 \times 3 \ so(3)$$ 矩阵且最后一行为 4 个 0 的 $$4 \times 4$$ 实矩阵的集合被称为 $$se(3)$$。 

#### (3) 刚体运动的指数坐标

定义齐次变换矩阵 $$T$$ 的六维指数坐标 $$S\theta \in \mathbb{R}^6$$，其中，$$S  \in \mathbb{R}^6$$ 为旋转轴，$$\theta \in \mathbb{R}$$ 为沿旋转轴将 $$I$$ 的原点移动到 $$T$$ 的原点的距离。

指数坐标 $$\mathcal{S\theta} \in \mathbb{R}^6$$ 的矩阵表示为：

$$
[S]=
\begin{bmatrix}
[\omega] & v \\
0 & 0
\end{bmatrix} \in se(3)
$$


矩阵指数 $$e^{[\mathcal{S}]\theta} \in SE(3)$$ 和 矩阵对数 $$[\mathcal{S}]\theta \in se(3)$$：

$$
exp: [\mathcal{S}]\theta \in se(3) \to T \in SE(3) \\
log: T \in SE(3) \to [\mathcal{S}]\theta \in se(3)
$$

如果 $$S_\omega=0,\Vert S_v \Vert=1$$：

$$
e^{[S]\theta}=
\begin{bmatrix}
I & \mathcal{S_v\theta} \\
0 & 1
\end{bmatrix}
$$

如果 $$\Vert S_\omega \Vert=1$$：

$$
e^{[S]\theta}=
\begin{bmatrix}
e^{[S_\omega]\theta} & (I\theta+(1-\cos\theta)[S_\omega]+(\theta-\sin\theta)[S_\omega]^2)S_v \\
0 & 1
\end{bmatrix}
$$

对于 {b} 通过 S 移动到 {b'}，如果 S 在 {b} 中表示：

$$
T_{sb'}=T_{sb}e^{[S_b]\theta}
$$

如果 S 在 {s} 中表示：

$$
T_{sb'}=e^{[S_s]\theta}T_{sb}
$$

#### (4) 力旋量

考虑作用在刚体上的一点 r 的纯力 f。定义参考坐标系 {a}，点 r 可表示为 $$r_a \in \mathbb{R}^3$$，力 f 可表示为 $$f_a \in \mathbb{R}^3$$。该力产生的力矩（torque）或力偶（moment）$$m_a \in \mathbb{R}^3$$ 可表示为：

$$
m_a = r_a \times f_a
$$

力旋量（wrench）：

$$
\mathcal{F}_a=
\begin{bmatrix}
m_a\\
f_a
\end{bmatrix} \in \mathbb{R}^6
$$

$$\mathcal{F_s}$$ 与 $$\mathcal{F_b}$$ 的关系推导：

$$
\begin{aligned}
\mathcal{V_s}^T\mathcal{F_s} &= \mathcal{V_b}^T\mathcal{F_b}=power\\
\mathcal{V_s}^T\mathcal{F_s} &= ([A_{dT_{bs}}]\mathcal{V_s})^T\mathcal{F_b}\\
\mathcal{V_s}^T\mathcal{F_s} &= \mathcal{V_s}^T[A_{dT_{bs}}]^T\mathcal{F_b}\\
\mathcal{F_s}&=[A_{dT_{bs}}]^TF_b
\end{aligned}
$$

#### (5) 测验

**1.** 已知 $$\hat{x}_a$$，$$\hat{y}_a$$，$$p_{sa}$$，可以通过图示法确定 $$\hat{z}_a$$，进而得到 $$T_{sa}$$：

$$
T_{sa}=
\begin{bmatrix}
0 & -1 & 0 & 0\\
0 & 0 & -1 & 0\\
1 & 0 & 0 & 1\\
0 & 0 & 0 & 1 
\end{bmatrix}
$$

```
[[0,-1,0,0],[0,0,-1,0],[1,0,0,1],[0,0,0,1]]
```

软件：

```python
import modern_robotics as mr
import numpy as np

R_sa=np.array([
    [0, -1, 0],
    [0, 0, -1],
    [1, 0, 0]
])
p_sa=np.array([0,0,1])
T_sa=mr.RpToTrans(R_sa,p_sa)
print (T_sa)
```

**2.**  已知 $$\hat{x}_b$$，$$\hat{y}_b$$，$$p_{sb}$$，可以通过图示法确定 $$\hat{z}_a$$，进而得到 $$T_{sb}$$：

$$
T_{sb}=
\begin{bmatrix}
1 & 0 & 0 & 0\\
0 & 0 & 1 & 2\\
0 & -1 & 0 & 0\\
0 & 0 & 0 & 1 
\end{bmatrix},
R_{sb}^T=
\begin{bmatrix}
1 & 0 & 0\\
0 & 0 & -1\\
0 & 1 & 0
\end{bmatrix},
-R_{sb}^Tp_{sb}=-
\begin{bmatrix}
1 & 0 & 0\\
0 & 0 & -1\\
0 & 1 & 0
\end{bmatrix}
\begin{bmatrix}
0 \\
2 \\
0
\end{bmatrix}
=
\begin{bmatrix}
0 \\
0 \\
-2
\end{bmatrix}
$$

进而：

$$
T_{sb}^{-1}=
\begin{bmatrix}
R_{sb}^T & -R_{sb}^Tp_{sb} \\
0 & 1
\end{bmatrix}
=
\begin{bmatrix}
1 & 0 & 0 & 0 \\
0 & 0 & -1 & 0 \\
0 & 1 & 0 & -2 \\
0 & 0 & 0 & 1
\end{bmatrix}
$$

```
[[1,0,0,0],[0,0,-1,0],[0,1,0,-2],[0,0,0,1]]
```

软件：

```python
import modern_robotics as mr
import numpy as np

R_sb=np.array([
    [1, 0, 0],
    [0, 0, 1],
    [0, -1, 0]
])
p_sb=np.array([0,2,0])
T_sb=mr.RpToTrans(R_sb,p_sb)
invT_sb=mr.TransInv(T_sb)
print (invT_sb)
```

**3.** 求 $$T_{ab}$$

$$
R_{sa}^T=
\begin{bmatrix}
0 & -1 & 0 \\
0 & 0 & -1 \\
1 & 0 & 0
\end{bmatrix}^T
=
\begin{bmatrix}
0&0&1\\
-1&0&0\\
0&-1&0
\end{bmatrix}
$$

$$
-R_{sa}^Tp_{sa}=-
\begin{bmatrix}
0&0&1\\
-1&0&0\\
0&-1&0
\end{bmatrix}
\begin{bmatrix}
0\\0\\1
\end{bmatrix}
=
\begin{bmatrix}
-1\\0\\0
\end{bmatrix}
$$

$$
\begin{aligned}
T_{ab}&=T_{as}T_{sb}=T_{sa}^{-1}T_{sb}\\
&=
\begin{bmatrix}
R_{sa}^T & -R_{sa}^Tp_{sa} \\
0 & 1
\end{bmatrix}T_{sb}\\
&=
\begin{bmatrix}
0&0&1&-1\\
-1&0&0&0\\
0&-1&0&0 \\
0 & 0 & 0 & 1
\end{bmatrix}
\begin{bmatrix}
1&0&0&0\\
0&0&1&2\\
0&-1&0&0 \\
0 & 0 & 0 & 1
\end{bmatrix}\\
&=
\begin{bmatrix}
0&-1&0&-1\\
-1&0&0&0\\
0&0&-1&-2 \\
0 & 0 & 0 & 1
\end{bmatrix}
\end{aligned}
$$

```
[[0,-1,0,-1],[-1,0,0,0],[0,0,-1,-2],[0,0,0,1]]
```

软件：

```python
import modern_robotics as mr
import numpy as np

R_sa=np.array([
    [0, -1, 0],
    [0, 0, -1],
    [1, 0, 0]
])
p_sa=np.array([0,0,1])
T_sa=mr.RpToTrans(R_sa,p_sa)
T_as=mr.TransInv(T_sa)
R_sb=np.array([
    [1, 0, 0],
    [0, 0, 1],
    [0, -1, 0]
])
p_sb=np.array([0,2,0])
T_sb=mr.RpToTrans(R_sb,p_sb)
T_ab=T_as.dot(T_sb)
print (T_ab)
```

**4.** $$T_1=TT_{sa}$$，左乘，是在固定坐标系 {s} 中表示的。

**5.** 已知 $$T_{sb}$$ 和 $$p_b=(1,2,3)^T$$，可得：

$$
\begin{bmatrix}
p_s\\
1
\end{bmatrix}
=T_{sb}
\begin{bmatrix}
p_{b}\\
1
\end{bmatrix}\\
=
\begin{bmatrix}
1&0&0&0\\
0&0&1&2\\
0&-1&0&0 \\
0 & 0 & 0 & 1
\end{bmatrix}
\begin{bmatrix}
1\\5\\-2\\1
\end{bmatrix}\\
=
\begin{bmatrix}
1\\5\\-2\\1
\end{bmatrix}\\
$$

```
[1,5,-2]
```

软件：

```python
import modern_robotics as mr
import numpy as np

R_sb=np.array([
    [1, 0, 0],
    [0, 0, 1],
    [0, -1, 0]
])
p_sb=np.array([0,2,0])
T_sb=mr.RpToTrans(R_sb,p_sb)
p_s=T_sb.dot(np.array([1, 2, 3, 1]))
print (p_s[0:3])
```

**6.** $$q=T_{sb}p_s$$，q 是不是 p 在 {b} 中的表示？

不是。

**7.** 已知 $$\mathcal{V_s}=(3,2,1,-1,-2,-3)^T$$，可得 $$\mathcal{V_a}$$：

$$
\mathcal{V_a}=[Ad_{T_{as}}]\mathcal{V_s}
=
\begin{bmatrix}
R_{as} & 0 \\
[p_{as}]R_{as} & R_{as} \\
\end{bmatrix}\mathcal{V_s}
=
\begin{bmatrix}
R_{sa}^T & 0 \\
[p_{as}]R_{sa}^T & R_{sa}^T
\end{bmatrix}\mathcal{V_s}
$$

还需计算 $$p_{as}$$，有两种方法，一是第3题计算出了 $$T_{as}$$，其中的向量即 $$p_{as}$$，二是由 $$T_{as}=T_{sa}^{-1}$$ 可以推导出：

$$
p_{as}=-R_{sa}^Tp_{sa}
$$

则：

$$
\mathcal{V_a}=
\begin{bmatrix}
R_{sa}^T & 0 \\
[-R_{sa}^Tp_{sa}]R_{sa}^T & R_{sa}^T
\end{bmatrix}\mathcal{V_s}
$$

里面的全是已知量，代入手算可得到结果：

```
[1,-3,-2,-3,-1,5]
```

软件：

```python
import modern_robotics as mr
import numpy as np

R_sa=np.array([
    [0, -1, 0],
    [0, 0, -1],
    [1, 0, 0]
])
p_sa=np.array([0,0,1])
T_sa=mr.RpToTrans(R_sa,p_sa)
T_as=mr.TransInv(T_sa)
AdT_as=mr.Adjoint(T_as)
V_s=np.array([3,2,1,-1,-2,-3])
V_a=AdT_as.dot(V_s)
print (V_a)
```

**8.** 根据教材 $$p_{53}$$ 的内容，已知 $$R_{sa}$$，可得：

$$
\theta=cons^{-1}(\frac{1}{2}(trR-1))=cos^{-1}(-\frac{1}{2})=\frac{2\pi}{3}\approx2.09
$$

```
2.09
```

**9.** 已知 $$\mathcal{S\theta}=(0,1,2,3,0,0)^T$$，求对应的矩阵指数。

$$
\mathcal{S\theta}=
\begin{bmatrix}
\omega \\ v
\end{bmatrix}\theta=
\begin{bmatrix}
\omega\theta \\ v\theta
\end{bmatrix}=
\begin{bmatrix}
0\\1\\2\\3\\0\\0
\end{bmatrix}
$$

先求解 $$\theta$$。已知 $$\omega\theta$$ 求 $$\theta$$ ，直接将 $$\omega\theta$$ 单位化后，计算 $$\theta$$ 即可。但是此处不能直接将 $$\mathcal{S\theta}$$ 整体单位化，需要根据具体情况单位化 $$\omega\theta$$ 或 $$v\theta$$：从 $$\mathcal{S\theta}$$ 可以看出 $$\omega \neq 0$$，因此，有 $$\Vert \omega \Vert=1$$，将 $$\omega$$ 单位化后可得：

$$
\theta=\sqrt{5}\approx{2.236}
$$

进而，可得：

$$
\omega=
\begin{bmatrix}
0\\\frac{1}{\sqrt{5}}\\\frac{2}{\sqrt{5}}
\end{bmatrix},
v=
\begin{bmatrix}
\frac{3}{\sqrt{5}}\\0\\0
\end{bmatrix},
[\omega]=
\begin{bmatrix}
0 & -\frac{2}{\sqrt{5}} & \frac{1}{\sqrt{5}}\\
\frac{2}{\sqrt{5}} & 0 & 0\\
-\frac{1}{\sqrt{5}} & 0 & 0
\end{bmatrix}
$$

代入下式计算即可（式中的 $$S_\omega$$ 即 $$\omega$$，$$S_v$$ 即 $$v$$）：

$$
\begin{aligned}
e^{[S]\theta}&=
\begin{bmatrix}
e^{[S_\omega]\theta} & (I\theta+(1-\cos\theta)[S_\omega]+(\theta-\sin\theta)[S_\omega]^2)S_v \\
0 & 1
\end{bmatrix}\\
&=
\begin{bmatrix}
I+sin\theta[\mathcal{S_\omega}]+(1-cos\theta)[\mathcal{S_\omega}]^2 & (I\theta+(1-\cos\theta)[S_\omega]+(\theta-\sin\theta)[S_\omega]^2)S_v \\
0 & 1
\end{bmatrix}
\end{aligned}
$$

软件：

```python
import modern_robotics as mr
import numpy as np

Stheta=np.array([0,1,2,3,0,0])
(S, theta)=mr.AxisAng6(Stheta)
se3mat=mr.VecTose3(Stheta)
T=mr.MatrixExp6(se3mat)
print (T)
```

结果（小数点后保留三位）：

```
[[-0.617,-0.704,0.352,1.056],[0.704,-0.294,0.647,1.941],[-0.352,0.647,0.677,-0.970],[0,0,0,1]]
```

**10.** 已知 $$T_{sb}$$ 和 $$\mathcal{F_b}=(1,0,0,2,1,0)^⊺$$，求 $$\mathcal{F_s}$$。

$$
[Ad_{T_sb}]=
\begin{bmatrix}
R_{sb} & 0\\
[p_{sb}]R_{sb} & R_{sb}
\end{bmatrix}
$$

第 3 题求解过程中计算出了 $$T_{sb}$$，从中取出 $$R_{sb}$$ 和 $$p_{sb}$$，代入计算即可。

软件：

```python
import modern_robotics as mr
import numpy as np

R_sb=np.array([
    [1, 0, 0],
    [0, 0, 1],
    [0, -1, 0]
])
F_b=np.array([1,0,0,2,1,0])
p_sb=np.array([0,2,0])
T_sb=mr.RpToTrans(R_sb,p_sb)
AdT_sb=mr.Adjoint(T_sb)
F_s=AdT_sb.dot(F_b)
print (F_s)
```

结果：

```
[1,0,0,2,0,-3]
```

**11.** 给定 $$T$$，使用 `TransInv` 求其逆矩阵。

```python
import modern_robotics as mr
import numpy as np

T=np.array([
    [0, -1, 0, 3],
    [1, 0, 0, 0],
    [0, 0, 1, 1],
    [0, 0, 0, 1]
])
invT=mr.TransInv(T)
print(T)
```

结果：

```
[[0,-1,0,3],[1,0,0,0],[0,0,1,1],[0,0,0,1]]
```

**12.** 给定运动旋量 $$\mathcal{V}=(1,0,0,0,2,3)^T$$，使用 `VecTose3` 求对应变换矩阵 T 的矩阵对数。

```python
import modern_robotics as mr
import numpy as np

V=np.array([1, 0, 0, 0, 2, 3])
se3mat=mr.VecTose3(V)
print(se3mat)
```

结果：

```
[[0,0,0,0],[0,0,-1,2],[0,1,0,3],[0,0,0,0]]
```

**13.** 已知 $$\hat{s}=(1,0,0), p=(0,0,2), h=1$$，使用 `ScrewToAxis` 求 $$\mathcal{S}$$。

```python
import modern_robotics as mr
import numpy as np

shat=np.array([1, 0, 0])
p=np.array([0, 0, 2])
h=1
S=mr.ScrewToAxis(p,shat,h)
print(S)
```

结果：

```
[1,0,0,1,2,0]
```

**14.** 已知矩阵对数（题目描述错了？给的条件实矩阵对数 $$[\mathcal{S}]\theta$$，不是矩阵指数 $$e^{[\mathcal{S}]\theta}$$） $$[\mathcal{S}]\theta$$，使用 `MatrixExp6` 求对应的变换矩阵 T。

```python
import modern_robotics as mr
import numpy as np

se3mat = np.array([[0,      -1.5708, 0, 2.3562 ],
                   [1.5708, 0,       0, -2.3562],
                   [0,      0,       0, 1      ],
                   [0,      0,       0, 0      ]])
T = mr.MatrixExp6(se3mat)
print(T)
```

结果(舍去小数部分)：

```
[[0,-1,0,3],[1,0,0,0],[0,0,1,1],[0,0,0,1]]
```

**15.** 已知 T，使用 `MatrixLog6` 计算对应的矩阵对数 $$[\mathcal{S}]\theta$$。

```python
import modern_robotics as mr
import numpy as np

T = np.array([
    [0, -1, 0, 3],
    [1, 0, 0, 0],
    [0, 0, 1, 1],
    [0, 0, 0, 1]
])
se3mat = mr.MatrixLog6(T)
print(se3mat)
```

结果（保留三位小数）：

```
[[0,-1.571,0,2.356],[1.571,0,0,-2.356],[0,0,0,1],[0,0,0,0]]
```