---
title: Introduction to Robotics-Stanford 笔记 LEC3
date: 2024-11-19 20:21:13
tags: 机器人学
banner_img: /images/bg/emt1.png
mathjax: true
---
<script src="https://fastly.jsdelivr.net/gh/misaka0502/live2d-widget@V0.2/autoload.js"></script>
<!-- <script src="/live2d-widget/autoload.js"></script> -->

没怎么听懂的一节，还得慢慢啃🤯

## Homogeneous Transform Interpretations

### Description of frame

$^{A}_{B}T:\{B\}=\{^{A}_{B}R\;\; {^{A}P_{Borg}}\}$

描述B相对于A的坐标系变换

### Transform mapping

$^{A}_{B}T:{^{B}P}\to{^{A}P}$

将坐标系B中的点P的描述映射到坐标系A中(点不变，向量变)

### Transform operator

$T:{P_{1}}\to{P_{2}}$

将坐标系A中的向量$P_{1}$变换成向量$P_{2}$(坐标系不变)

## Transform Equation

### Compound Transformations 复合变换

引入第三个坐标系C，假设有一个从C到B的变换$^{B}_{C}T$和一个从B到A的变换$^{A}_{B}T$，要研究从C到A的变换

假设C中有一个向量$^{C}P$：

$$
{^{B}P={^{B}_{C}T}{^{C}P}}\\
{^{A}P={^{A}_{B}T}{^{B}P}}\\
{^{A}P={^{A}_{B}T}{^{B}_{C}P}^{C}P} \implies {^{A}_{C}T={^{A}_{B}T}{^B_CT}}
$$

- 左上角保持相对于旋转的结构，有相同的旋转属性
- 右上角可以看作是将C的原点相对于B的向量映射到A中

### Transform Equation

假设有A, B, C, D四个坐标系，A→B→C→D→A这样的复合变换，构成一个闭环的变换组，最终的总变换矩阵是一个单位阵，即

$$
{^A_BT}{^B_CT}{^C_DT}{^D_AT}=I
$$

假设这四个变换其中有一个是未知的，就可以通过以上恒等式求解出来，例如：

$$
^B_AT={^B_CT}{^C_DT}{^D_AT}
$$

# Representations

> 我们所关心的问题在于末端执行器，末端执行器实际是整个操纵问题的目的，即我们关心如何在空间中定位这个末端执行器，以及如何将其移动到某个位置
> 

假设从Base frame基坐标系到末端执行器坐标系的齐次变换是$^B_ET$，当完成正向运动学时，末端执行器相对于基坐标系的变换由$^B_ET$给出，并且这个$^B_ET$来自整个操纵器的描述，包括连杆长度、倾斜角度、连杆特性等，并且它是**关节角度的函数**（核心），即$^B_ET$会随着关节角度变化而变化，机器人配置（configuration）会影响末端执行器位置。所以我们要找到这个4x4矩阵T作为关节角度的函数，这个矩阵将包含每个配置的齐次变换的描述。

> 所以我们需要做的是从T中提取位置和方向的描述
> 

## End-Effector Configuration Parameters

$$
X=\begin{bmatrix}
X_P \\
X_R
\end{bmatrix}
$$

其中$X_P$代表位置（x,y,z），$X_R$代表方向。回忆一下齐次变换矩阵，可以发现

- $X_P$的信息在T的最后一列的前三个元素构成的向量里
- $X_R$的信息在T左上角的3x3子矩阵里，即旋转矩阵

## Position Representations 位置表示

![](/images/lec3-1.png)

- Cartesian 笛卡尔坐标：$(x,y,z)$
- Cylindrical 圆柱坐标：$(\rho,\theta,z)$
- Spherical 球坐标：$(r,\theta,\phi)$

> 大多数情况下可以使用笛卡尔坐标；如果希望操纵工具沿轴平移进行灵巧操作，可能会使用圆柱坐标。对于不同的任务，使用不同的位置表示可能会更有优势。
> 

## Rotation Representations 旋转表示

### Rotation Matrix 旋转矩阵

- 包含旋转中的所有信息的表示

$$
R=\begin{bmatrix}
r_{11} & r_{12} & r_{13} \\
r_{21} & r_{22} & r_{23} \\
r_{31} & r_{32} & r_{33}
\end{bmatrix}=[\boldsymbol{r_1} \;\; \boldsymbol{r_2} \;\; \boldsymbol{r_3}]
$$

从齐次变换T中可以提取这个旋转矩阵。回忆一下旋转矩阵，$\boldsymbol{r_1},\boldsymbol{r_2},\boldsymbol{r_3}$实际是X, Y, Z在基座标系中的分量，现在我们可以用它构建一个表示

### Direction Cosines 方向余弦

$$
\boldsymbol{x_r}=\begin{bmatrix}
\boldsymbol{r_1} \\
\boldsymbol{r_2} \\
\boldsymbol{r_3}
\end{bmatrix}_{9\times1}
$$

> 上面两种表示都需要9个参数，而表示方向的自由度实际上只有3个，我们可以考虑以下这些向量之间的约束
> 

### Constrains 约束

$\boldsymbol{r_1},\boldsymbol{r_2},\boldsymbol{r_3}$都是模长为1的单位向量，且$\boldsymbol{r_1},\boldsymbol{r_2},\boldsymbol{r_3}$之间两两正交，这里一共有3+3=6个约束

$$
|\boldsymbol{r_1}|=|\boldsymbol{r_2}|=|\boldsymbol{r_3}|=1\\
\boldsymbol{r_1}\cdot\boldsymbol{r_2}=\boldsymbol{r_1}\cdot\boldsymbol{r_3}=\boldsymbol{r_2}\cdot\boldsymbol{r_3}=0
$$

因此9个参数-6个约束=3个自由度

> 实际上旋转矩阵的方向余弦都是**冗余表示**，即参数并不是独立的，这会产生一个问题。
假设要让机器人做一个移动物品的简单任务，从一个点移动到另一个点，一般可以采用插值的方法，在起点和终点之间插入一系列点，由于约束的存在，就必须时时刻刻监视约束是否满足，这是非常困难的
> 

### Three Angle Representations

- **Fixed Angles 固定角**，即绕固定（参考坐标系）轴转

> 首先将坐标系{B}和一个已知的参考坐标系{A}重合。先将{B}绕$\hat{X_A}$旋转γ角，再绕$\hat{Y_A}$旋转β角，最后绕$\hat{Z_A}$旋转α角。由于每个旋转都是绕着固定参考坐标系{A}的轴，故称这种姿态表示法为X-Y-Z固定角。有时把它们定义为**回转角、俯仰角和偏转角（roll、pitch、yaw）**
> 
> 
> ![](/images/lec3-2.png)
> 

可直接推导等价旋转矩阵：

$$
^A_BR_{XYZ}=R_Z(\alpha).R_Y(\beta).R_X(\gamma)\\
=\begin{bmatrix}
c\alpha & -s\alpha & 0 \\
s\alpha & c\alpha & 0 \\
0 & 0 & 1
\end{bmatrix}
\begin{bmatrix}
c\beta & 0 & s\beta \\
0 & 1 & 0 \\
-s\beta & 0 & c\beta
\end{bmatrix}
\begin{bmatrix}
1 & 0 & 0 \\
0 & c\gamma & -s\gamma \\
0 & s\gamma & c\gamma
\end{bmatrix}
$$

- **Euler Angles (Z-Y-X)**：Z-Y-X 欧拉角

> 首先将坐标系{B}和一个已知的参考坐标系{A}重合。先将{B}绕$\hat{Z_B}$转α角，再绕$\hat{Y_B}$绕β角，最后绕$\hat{X_B}$转γ角。在这种表示法中，每次都是绕运动坐标系{B}的各轴旋转而不是绕固定坐标系{A}的各轴旋转。这样三个一组的旋转被称作**欧拉角**。
**注意每次旋转所绕的轴的姿态取决于上一次的旋转。**
> 
> 
> ![](/images/lec3-3.png)
> 

假设从坐标系A到B’，B’到B’’，B‘’到B的旋转矩阵分别是$^A_{B'}R,^{B'}_{B''}R,^{B''}_BR$，显然

$$
^A_BR=^A_{B'}R.^{B'}_{B''}R.^{B''}_{B}R
$$

也可以写成

$$
^A_BR=R_Z(\alpha).R_Y(\beta).R_X(\gamma)\\
=\begin{bmatrix}
c\alpha & -s\alpha & 0 \\
s\alpha & c\alpha & 0 \\
0 & 0 & 1
\end{bmatrix}
\begin{bmatrix}
c\beta & 0 & s\beta \\
0 & 1 & 0 \\
-s\beta & 0 & c\beta
\end{bmatrix}
\begin{bmatrix}
1 & 0 & 0 \\
0 & c\gamma & -s\gamma \\
0 & s\gamma & c\gamma
\end{bmatrix} \\
=\begin{bmatrix}
c\alpha c\beta & X & X \\
s\alpha c\beta & X & X \\
-s\beta & c\beta s\gamma & c\beta c\gamma
\end{bmatrix}
$$

**注意这个结果与以相反顺序绕固定轴旋转三次得到的结果完全相同。即三次绕固定轴旋转的最终姿态和以相反顺序绕运动坐标轴转动的最终姿态相同。**

$$
R_{Z'Y'X'}(\alpha,\beta,\gamma)=R_{XYZ}(\gamma,\beta,\alpha)
$$

其中X是一些复杂的式子，但我们不关心，因此可以忽略。

至此得到了一个以转角α、β、γ为参数的旋转矩阵

- Z-Y-Z Euler Angles

懒得写了，还有很多不同转角组合，一共有24中表示法，称作**转角排列设定法**，其中12种为固定角设定法，另12种为欧拉角设定法。

### Example

![](/images/lec3-4.png)

### Inverse Problem

逆问题即从旋转矩阵等价推出X-Y-Z固定角

$$
^A_BR=R_Z(\alpha).R_Y(\beta).R_X(\gamma)\\
=\begin{bmatrix}
c\alpha & -s\alpha & 0 \\
s\alpha & c\alpha & 0 \\
0 & 0 & 1
\end{bmatrix}
\begin{bmatrix}
c\beta & 0 & s\beta \\
0 & 1 & 0 \\
-s\beta & 0 & c\beta
\end{bmatrix}
\begin{bmatrix}
1 & 0 & 0 \\
0 & c\gamma & -s\gamma \\
0 & s\gamma & c\gamma
\end{bmatrix} \\
=\begin{bmatrix}
c\alpha c\beta & X & X \\
s\alpha c\beta & X & X \\
-s\beta & c\beta s\gamma & c\beta c\gamma
\end{bmatrix}
$$

通过三角函数关系可以算出：

![](/images/lec3-5.png)

如果cosβ=0（β=±90°），以上的求解就不能成立，这样的点成为奇点（Singularity of the representation）。在这种情况下，只能求出α和γ的和或差，无法区分α和γ。在这种情况下一般取α=0

Example：

![](/images/lec3-6.png)

> 在奇点处将无法计算与α相关的速度，无法跟踪其运动
任何三参数的角度表示法都会遇到奇点问题，而方向余弦这种9参数的表示法没有奇点问题，但是有冗余
> 

### Equivalent angle-axis representation 等效角度-轴线表示 $R_K(\theta)$

可以证明，总是存在一个向量$\boldsymbol{K}$，能够使坐标系A绕着向量$\boldsymbol{K}$旋转一定角度到坐标系B

首先将坐标系{B}和一个已知的参考坐标系{A}重合。将{B}绕矢量$^{A}\hat{\boldsymbol{K}}$按右手定则转θ角。

$^{A}\hat{\boldsymbol{K}}$是一个长度为1的单位向量，因此确定它只需要两个参数，角度确定了第三个参数。

经常用旋转量θ乘以单位方向矢量$\hat{\boldsymbol{K}}$形成一个简单的3x1矢量来描述姿态，即

$$
X_r=\theta.K=\begin{bmatrix}
\theta.k_X \\
\theta.k_Y \\
\theta.k_Z
\end{bmatrix}
$$

当选择{A}的主轴的其中一个作为旋转轴时，则等效旋转矩阵就变成了之前的平面旋转矩阵

若旋转轴为一般轴，则等效旋转矩阵为

![](/images/lec3-7.png)

其中vθ=1-cosθ

> 从一个给定的旋转矩阵求$\hat{\boldsymbol{K}}$和θ，这里直接给出结果，计算过程见书
> 

$$
^{A}_{B}R_K(\theta)=\begin{bmatrix}
r_{11} & r_{12} & r_{13} \\
r_{21} & r_{22} & r_{23} \\
r_{31} & r_{32} & r_{33}
\end{bmatrix} \\
\theta=Acos(\frac{r_{11}+r_{22}+r_{33}-1}{2}) \\
\hat{K}=\frac{1}{2sin\theta}\begin{bmatrix}
r_{32}-r_{23} \\
r_{13}-r_{31} \\
r_{21}-r_{12}
\end{bmatrix}
$$

由于这里sinθ在分母上，还是会有奇点问题

### Euler Parameters 欧拉参数

欧拉参数通过4个数值表示姿态

![](/images/lec3-8.png)

> 给定旋转矩阵，得到对应的欧拉参数
> 
> 
> ![](/images/lec3-9.png)
> 
- 引理：对于所有旋转，总是有一个欧拉参数大于或等于二分之一

虽然这里$\epsilon_4$在分母上，但是欧拉参数表示法没有奇点问题

对于所有的$\epsilon_i$，其值在[-1, 1]区间