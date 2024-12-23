---
title: Introduction to Robotics-Stanford 笔记 LEC4
date: 2024-12-23 20:16:05
tags: Robotics
mathjax: true
banner_img: /images/bg/qiannian1.jpg
---
<script src="https://fastly.jsdelivr.net/gh/misaka0502/live2d-widget@V0.2/autoload.js"></script>
<!-- <script src="/live2d-widget/autoload.js"></script> -->

# LEC4 Manipulator Kinematics 机械臂运动学

> 正运动学是一种随动坐标系与基坐标系之间的关系
> 

## Link Description 连杆描述

![1](/images/introduction-to-robotics-lec4/image.png)

对于一个连杆，其两端有两个关节轴，如何描述这两个轴的关系？为了去确定机械臂两个相邻关节轴的位置关系，可把连杆看作一个刚体。用空间中的直线来表示关节轴。显然，在描述连杆的运动时，**一个连杆运动可用两个参数描述，这两个参数定义了空间中两个关节轴之间的相对位置（距离和夹角）**

- $\overrightarrow{a}_{i-1}$：Link Length **连杆长度**
    - mutual perpendicular 两轴之间公垂线的长度
    - unique except for parallel axis 是唯一的（平行轴例外）
    - $\overrightarrow{a}_{i-1}$**是常数**
- $\alpha_{i-1}$：Link Twist **连杆扭转角**
    - measured in the right-hand sense about $\overrightarrow{a}_{i-1}$ 假设一与公垂线$\overrightarrow{a}_{i-1}$垂直的平面，将轴$i$和轴$i-1$投影到平面上，用右手定则从轴$i-1$绕$\alpha_{i-1}$到轴$i$测量两轴线之间的夹角
    - $\alpha_{i-1}$**是常数**

一种特殊情况是Intersecting Joint Axes（两个关节轴相交），在这种情况下，考虑轴$i-1$和轴$i$形成的平面，并且做出该平面的垂线，就会得到两个轴的公垂线向量。注意$\overrightarrow{a}_{i-1}$的方向会影响$\alpha_{i-1}$的方向。**一般可以让$\overrightarrow{a}_{i-1}$指向末端执行器。**

## Link Connection 连杆连接的描述

![image.png](/images/introduction-to-robotics-lec4/image%201.png)

描述连杆连接时，同样只需要两个参数，这两个参数完全确定了所有连杆是怎么连接的。

### 处于运动链中间位置的连杆

- $d_i$：Link Offset **连杆偏距**
    - variable if joint $i$ is prismatic
    - $d_i$对于转动关节来说是固定不变的，但对于平移关节，就会沿着移动方向影响后续连杆的运动。**如果是平移关节，那么$d_i$是变量。**
- $\theta_i$：Joint Angle **关节角**
    - variable if joint $i$ is revolute
    - **对于一个转动关节，$\theta_i$是变量**

$d_i$和$\theta_i$之间总有一个是常数，另一个是变量。（取决于关节类型）

### 运动链中首端和末端连杆

首先是连杆长度和连杆扭转角。连杆长度$a_i$和连杆扭转角$\alpha_i$取决于关节轴线$i$和$i+1$，因此轴1到n确定了$a_1,a_2...a_{n-1}$和$\alpha_1,\alpha_2...\alpha_{n-1}$。对于首端和末端连杆，其参数习惯设定为$a_0=a_n=0$和$\alpha_0=\alpha_n=0$，相当于把轴0和轴1放在同一个坐标轴上重合，这样可以减少正运动学中使用的参数量。

其次是连杆偏距和关节角。注意，$\theta_i$和$d_i$取决于连杆i-1和i，这意味着我们实质上是定义$\theta_2,\theta_3...\theta_{n-1}$和$d_2,d_3...d_{n-1}$。**如果关节1为转动关节，则$\theta_1$的零位可以任意选取，并且设定$d_1=0$。同样，如果关节1为移动关节，则$d_1$的零位可以任意选取，并且设定$\theta_1=0$。**

以上方法也适用于关节n。

## 连杆参数 Denavit-Hartenberg Parameters

由上可知，机器人的每个连杆都可以用4个运动学参数来描述，其中**两个参数用于描述连杆本身($\alpha_i,a_i$)，另两个参数用于描述连杆之间的连接关系($d_i,\theta_i$)**。通常，对于转动关节，$\theta_i$为**关节变量**，其他三个**连杆参数**是固定不变的；对于移动关节，$d_i$为关节变量，其他三个连杆参数是固定不变的。这种用连杆参数描述机构运动关系的规则称为Denavit-Hartenberg方法，即

$$
4 \, D-H \, parameters (\alpha_i,a_i,d_i,\theta_i)
$$

例如，对于一个6关节机器人，需要用18个参数就可以完全描述这些**固定的运动学参数**。如果是6个转动关节的机器人，这时18个固定参数可以分为6组$(a_i,\alpha_i,d_i)$表示。

## 连杆坐标系的定义

为了描述每个连杆与相邻连杆之间的相对位置关系，需要在每个连杆上定义一个固连坐标系。根据固连坐标系所在连杆的编号对固连坐标系命名，因此，固连在连杆$i$上的固连坐标系称为坐标系$\{i\}$。

### 运动链中间位置连杆坐标系的定义

通常按照如下方法确定连杆上的固连坐标系：坐标系$\{i\}$的$Z$轴称为$Z_i$，并于关节轴$i$重合，坐标系$\{i\}$的原点位于公垂线$a_i$与关节轴$i$的交点处。$X_i$沿$a_i$方向由关节$i$指向关节$i+1$。

当$a_i=0$时（相交轴），$X_i$垂直于$Z_i$和$Z_{i+1}$所在的平面。按右手定则绕$X_{i}$轴的转角定义为$\alpha_i$，由于$X_i$轴的方向可以由两种选择，因此$\alpha_i$的符号也有两种选择。$Y_i$轴由右手定则确定，从而完成了对坐标系$\{i\}$的定义。

![image.png](/images/introduction-to-robotics-lec4/image%202.png)

### 运动链中首端连杆和末端连杆坐标系的定义

固连于基座（即连杆0）上的坐标系为坐标系$\{0\}$。这个坐标系是一个固定不动的坐标系。

参考坐标系$\{0\}$可以任意设定，但是为了使问题简化，**通常设定$Z_0$轴沿关节轴1的方向，并且当关节变量1为0时，设定参考坐标系$\{0\}$与坐标系$\{1\}$重合。**

- 首端连杆
    - 当关节1为转动关节时，$a_0=\alpha_0=d_0=0$，当$\theta_1=0$时，$\{0\} \equiv \{1\}$
    - 当关节1为移动关节时，$a_0=\alpha_0=\theta_0=0$，当$d_1=0$时，$\{0\} \equiv \{1\}$
- 末端连杆
    - 当关节n为转动关节时，$a_n=\alpha_n=d_n=0$，当$\theta_n=0$时，$\{n\} \equiv \{n-1\}$
    - 当关节n为移动关节时，$a_n=\alpha_n=\theta_n=0$，当$d_n=0$时，$\{n\} \equiv \{n-1\}$

总结一下四个参数：

- $a_i$: distance $(z_i,z_{i+1})$along $x_i$
- $\alpha_i$: angle $(z_i,z_{i+1})$ about $x_i$
- $d_i$: distance $(x_{i-1},x_i)$ along $z_i$
- $\theta_i$: angle $(x_{i-1},x_i)$ about $z_i$

例子：

![image.png](/images/introduction-to-robotics-lec4/image%203.png)

## 机械臂运动学 Forward Kinematics

### 连杆变换的推导

描述连杆的四个参数分别对应一个简单的坐标系变换，一共四个坐标系变换。

我们希望建立坐标系$\{i\}$相对于坐标系$\{i-1\}$的变换。一般这个变换是由4个连杆参数构成的函数。对任意给定的机器人，这个变换是只有一个变量的函数，另外3个参数由机械系统确定。**通过对每个连杆逐一建立坐标系，我们把运动学问题分解为n个子问题**。为解决每个子问题，即$^{i-1}_{\quad i}T$，我们将每个子问题再分解为4个次子问题。4个变换中的每一个变换都是仅有一个连杆参数的函数。为每个连杆定义3个中间坐标系——$\{P\}、\{Q\}和\{R\}$。

![image.png](/images/introduction-to-robotics-lec4/image%204.png)

由于旋转 $α_{i-1}$，因此坐标系 $\{R\}$ 与坐标系 $\{i-1\}$ 不同；由于位移 $a_{i-1}$，因此坐标系 $\{Q\}$ 与坐标系 $\{R\}$ 不同；由于位移 $a_i$，因此坐标系 $\{P\}$ 与坐标系 $\{Q\}$ 不同；由于转角 $𝜃_𝑖$   ，因此坐标系 $\{P\}$ 与坐标系 $\{Q\}$ 不同；由于位移  $d_i$  ，因此坐标系 $\{i\}$ 与坐标系 $\{P\}$ 不同。

整个过程可以写成

$$
_i^{i-1}T=_R^{i-1}T_Q^RT_P^QT_i^PT
$$

考虑每一个变换矩阵，上式可以写成

$$
_i^{i-1}T=R_X(a_{i-1})D_X(a_{i-1})R_Z(\theta_i)D_Z(d_i)
$$

计算可得

$$
_i^{i-1}T=\begin{bmatrix}c\theta_i & -s\theta_i & 0 & a_{i-1} \\s\theta_ic\alpha_{i-1} & c\theta_ic\alpha_{i-1} & -s\alpha_{i-1} & -s\alpha_{i-1}d_i \\s\theta_is\alpha_{i-1} & c\theta_is\alpha_{i-1} & c\alpha_{i-1} & c\alpha_{i-1}d_i \\0 & 0 & 0 & 1\end{bmatrix}
$$

结合上面例子中给出的参数表，就可以写出整个变换矩阵

### 连杆变换的连乘

如果已经定义了连杆坐标系和相应的连杆参数，就能直接建立运动学方程。由连杆参数值，我们可以计算出各个连杆变换矩阵。把这些连杆变换矩阵连乘就能得到一个坐标系$\{N\}$相对于坐标系$\{0\}$的变换矩阵

$$
_N^0T=_1^0T_2^1T_3^2T\cdots_N^{N-1}T
$$

变换矩阵$_{N}^0T$是关于$n$个关节变量的函数。如果能得到机器人关节位置传感器的值，机器
人末端连杆在笛卡儿坐标系里的位置和姿态就能通过$_N^0T$计算出来。