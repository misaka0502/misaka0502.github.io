---
title: 多智能体强化学习(MARL)值函数分解——从VDN到QMIX
date: 2024-12-07 22:48:34
tags: 
    - 强化学习
    - 多智能体
mathjax: true
banner_img: /images/bg/emt1.png
---
<script src="https://fastly.jsdelivr.net/gh/misaka0502/live2d-widget@V0.2/autoload.js"></script>
<!-- <script src="/live2d-widget/autoload.js"></script> -->

# 一、MARL中的难点

## 1、部分可观察

当${agents}$和环境进行交互时，${agents}$无法看到和环境的全局状态$\mathbf{s}$，只能观察到自己视野范围内的局部信息$\mathbf{o}$。

比如StarCraft II：

> Partial observability is achieved by the **introduction of unit sight range**, which restricts the agents from receiving information about allied or enemy units that are out of range. Moreover, agents can only observe others if they are alive and cannot distinguish between units that are dead or out of range.
> 

## 2、不稳定性

在multi-agent环境中，由于${agents}$之间相互影响，因此${agent}_i$在观察$o_i$下执行动作$u_i$后得到的

$r_i$与$o'_i$是所有${agents}$的行为共同造成的，即$r_i=R_i(\bf{s},\bf{u}),o'_i=T_i(\bf{s},\bf{u})$。那么此时对于${agent}_i$而言，即使它在观察$o_i$下一直都执行动作$u_i$，但是由于$\mathbf{s}$未知且其他${agents}$的策略在不断变化，此时${agent}_i$得到的$r_i$和$o'_i$可能是不同的。这就是MARL中的不稳定性，即reward与transition存在不稳定性，从而导致${agent}_i$的值函数$Q_i(o_i,u_i)$的更新就会十分不稳定。

总结造成不稳定性的原因主要是两点：

- 部分可观察的场景使得$o_i$下对应的$\mathbf{s}$有很多
- 其他${agents}$也在学习，策略不断在变化，选择的动作也在不断变化

因此，同样的$(o_i,u_i)$可能对应着许多不同的$(\bf{s},\bf{u})$，从而导致其获得的反馈不稳定。

## 二、为什么要进行值函数分解

主要的原因就在于，每个智能体只能在自己的角度去观测和决策，无法站在全局的角度去观测和决策，从而无法学到全局的最优策略。为了解决这个问题，学者提出使用𝐶𝑒𝑛𝑡𝑟𝑎𝑙𝑖𝑧𝑒𝑑 𝑇𝑟𝑎𝑖𝑛𝑖𝑛𝑔 𝐷𝑒𝑐𝑒𝑛𝑡𝑟𝑎𝑙𝑖𝑧𝑒𝑑 𝐸𝑥𝑐𝑢𝑡𝑖𝑜𝑛(𝐶𝑇𝐷𝐸)的方法，将条件限制放松，允许${agents}$在训练的时候可以访问全局信息，从而站在全局的角度去训练。但是即使能站在全局角度去训练，要训练出一个什么形式的策略才行？

一个直观的想法是去训练一个全局的$Q_{total}(\bf{s},\bf{u})$，它考虑了全局信息，可以直接客服MARL中的不稳定性。新的问题在于，部分可观察约束注定了$Q_{total}(\bf{s},\bf{u})$就算训练出来了也用不上。

综上，仅仅使用${agents}$的$Q_i(o_i,u_i)$进行决策存在不稳定性，只有$Q_{total}(\bf{s},\bf{u})$才能站在全局的角度进行学习去解决不稳定性，但得到了$Q_{total}(\bf{s},\bf{u})$又无法直接使用，因此就出现了一系列值函数分解的方式来解决这个问题。

# 三、VDN（Value Decomposition Networks）

VDN的思想很简单，就是将$Q_{total}(\bf{s},\bf{u})$视为$Q_i(o_i,u_i)$的总和，即：

$$
Q_{total}(s,u)=\sum_{i=1}^NQ_i(o_i,u_i)
\tag{1}
$$

其中$N$是环境中agents的数量。近似得到$Q_{total}(\bf{s},\bf{u})$之后，VDN使用DQN的方法进行更新，通过全局奖励R来更新$Q_{total}(\bf{s},\bf{u})$，优化目标为：

$$
L(\theta)=\frac{1}{M}\sum_{j=1}^M\left(y_j-Q_{total}(s,u)\right)^2 \tag{2}
$$

其中$y_j=r_j+\max_u\hat{Q}_{total}\left(\bf{s'},u\right)$，$\hat{Q}_{total}=\sum_{i=1}^N\hat{Q}_i(o_i,u_i)$，$\hat{Q}_i$是目标网络。

通过这种方法，梯度就会通过$Q_{total}$反向传递给每个$Q_i$，这样就可以站在全局的角度去更新它们。

# 四、QMIX（Monotonic Value Function Factorisation for Deep Multi-Agent Reinforcement Learning）

## 1、VDN的缺点

VDN是将所有的$Q_i(o_i,u_i)$加起来去近似$Q_{total}(\bf{s},\bf{u})$，即认为$Q_i(o_i,u_i)$和$Q_{total}(\bf{s},\bf{u})$的关系是求和，通过累加和来近似$Q_{total}(\bf{s},\bf{u})$。但是这个关系太简单了，不能表达复杂的$Q_{total}(\bf{s},\bf{u})$。此外，VDN已经采用了CTDE，集中式训练的优势没有完全发挥出来。

## 2、QMIX的思想

QMIX也是基于CTDE和DQN的方法，其与VDN最主要的区别就在于QMIX是利用一个神经网络（称为“**混合网络**”）去近似$Q_{total}$，以表达$Q_i(o_i,u_i)$和$Q_{total}(\bf{s},\bf{u})$之间更为复杂的关系。

> Key to our method is the insight that the full factorisation of VDN is not necessary in order to be able to extract decentralised policies that are fully consistent with their centralised counterpart. Instead, for consistency we only need to ensure that a global argmax performed on $Q_{total}$ yields the same result as a set of individual argmax operations performed on each $Q_i$
我们方法的关键是认识到，为了能够提取与集中式策略完全一致的去中心化策略，不需要对 VDN 进行完全分解。相反，为了保持一致性，**我们只需要确保在**$Q_{total}$**上执行的全局 argmax 产生与在每个** $Q_i$**上执行的一组单独的 argmax 操作相同的结果**
> 

由此引出QMIX中**最核心的一个约束**：

$$
\frac{\partial Q_{tot}}{\partial Q_a}\geq0,\forall a\in A. \tag{3}
$$

这也是论文题目中**“Monotonic”**一词的体现，通过这个约束来保证$Q_{total}(\bf{s},\bf{u})$对$Q_i(o_i,u_i)$是单调的，从而保证每个 agent 选择的**局部最优动作**$\operatorname{argmax}_{u_i}Q_i(o_i,u_i)$恰好就是**全局最优动作**$\operatorname{argmax}_{u}Q_{total}(s,u)$的一部分（其实就是单调函数的性质？），即：

$$
\operatorname{argmax}_{\mathbf{u}}Q_{tot}(\tau,\mathbf{u})=\begin{pmatrix}\operatorname{argmax}_{u^1}Q_1(\tau^1,u^1) \\\vdots \\\operatorname{argmax}_{u^n}Q_n(\tau^n,u^n)\end{pmatrix} \tag{4}
$$

而这个约束是通过限制混合网络的每层权重$\bf{w}$大于0实现的（偏差$\bf{b}$不用限制）。

### 2.1 **算法大框架 —— 基于 AC 框架的 CTDE（Centralized Training Distributed Execution） 模式**

![image.png](/images/qmix_image.png)

多智能体强化学习（MARL）训练中面临的最大问题是：训练阶段和执行阶段获取的信息可能存在不对等问题。即，在训练的时候我们可以获得大量的全局信息（事实证明，只有获取足够的信息模型才能被有效训练）。

但在最终应用模型的时候，我们是无法获取到训练时那么多的全局信息的，因此，人们提出两个训练网络：一个为中心式训练网络（Critic），该网络只在训练阶段存在，获取全局信息作为输入并指导 Agent 行为控制网络（Actor）进行更新；另一个为行为控制网络（Actor），该网络也是最终被应用的网络，在训练和应用阶段都保持着相同的数据输入。

AC 算法的应用非常广泛，QMIX 在设计时同样借鉴了 AC 的 “中心式网络” 和 “分布式执行器” 的想法，整个网络包含了 Mixing Network（类比 Critic 网络）和 Agent RNN Network（类比 Actor 网络）

### 2.2 **Agent RNN Network**

QMIX 中每一个 Agent 都由 RNN 网络控制，在训练时可以为每一个 Agent 个体都训练一个独立的 RNN 网络，同样也可以所有 Agent 复用同一个 RNN 网络。

RNN 网络一共包含 3 层，输入层（MLP）→ 中间层（GRU）→ 输出层（MLP）

![image.png](/images/qmix_image1.png)

### 2.3 混合网络

混合网络同时接收 $Q_i(o_i,u_i)$和当前全局状态$\mathbf{s}$ ，输出在当前状态下所有 ${agents}$联合行为$\bf{u}$ 的$Q_{total}(\bf{s},\bf{u})$。

![image.png](/images/qmix_image2.png)

混合网络使用**前馈神经网络结构。**

前面说到，混合网络要满足式（3），如果𝒔和$Q_i$一起输入到混合网络，会引入不必要的约束，因此混合网络**使用 hypernetworks 去利用全局状态**𝒔。
上图中蓝色部分（中间层神经元）的权重（weights）和偏差（bias）均由**hypernetworks**产生。即，混合网络中实际包含两个神经网络，红色参数生成网络 & 蓝色推理网络。

- **hypernetworks**： 接收全局状态$\mathbf{s}$ ，生成蓝色网络中的神经元权重（weights）和偏差（bias）。
- **推理网络**：接收所有 Agent 的行为效用值  $Q_i$，并将参数生成网络生成的权重和偏差赋值到网络自身，从而推理出全局$Q_{total}(\bf{s},\bf{u})$。

在这里可以看到**hypernetworks**输出权重的时候会加一个绝对值使权重不小于0，这就是前面说的为了保证式（3）成立，保证**局部最优动作**就是**全局最优动作**。

### 2.4 **模型更新流程**

本质上和VDN是一样的，都是根据DQN的方法去优化$\mathcal{L}(\theta)=\sum_{i=1}^b\left[\left(y_i^{tot}-Q_{tot}(\tau,\mathbf{u},s;\theta)\right)^2\right]$.

# 参考文献

[**[伏羲讲堂]多智能体强化学习中的值函数分解——VDN、QMIX、QTRAN**]([https://zhuanlan.zhihu.com/p/203164554](https://zhuanlan.zhihu.com/p/203164554))

[**【QMIX】一种基于Value-Based多智能体算法**]([https://zhuanlan.zhihu.com/p/353524210](https://zhuanlan.zhihu.com/p/353524210))