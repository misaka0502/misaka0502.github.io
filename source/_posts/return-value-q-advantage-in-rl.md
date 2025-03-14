---
title: 强化学习中回报（Return）、价值（Value）、动作价值（Action-Value）和优势（Advantage）的联系
date: 2025-03-14 16:17:00
tags: 
    - 强化学习
    - 深度学习
mathjax: true
banner_img: /images/bg/haoduo.jpg
---
<script src="https://fastly.jsdelivr.net/gh/misaka0502/live2d-widget@V0.2/autoload.js"></script>
<!-- <script src="/live2d-widget/autoload.js"></script> -->

在做项目的过程中总是对这些概念有些模糊，如果不搞明白很容易在读论文和看代码的时候绕晕，因此写一篇笔记理清一下关系。

# 1.回报 Return

回报$G_{t}$表示从时间步$t$开始，经过未来所有时间步（或是有限步数）的折扣累积奖励。形式上可以写为：

$$
G_t=R_t+\gamma R_{t+1}+\gamma^2R_{t+2}+\cdots=\sum_{k=0}^\infty\gamma^kR_{t+k}
$$

其中：

- $R_{t}$是在时间步$t$的即时奖励
- $\gamma\in[0,1]$是折扣因子，用于衡量未来奖励的重要性
- $T$是终止时间步（在无限时间步的情形下，使用极限）

回报$G_{t}$代表了**从时间步$t$开始智能体可以获得的总收益。**

# 2.值函数 Value Function

状态值函数$V_{\pi}(s_{t})$是在状态$s_{t}$下，期望的回报。它表示在当前状态下，如果按照策略$\pi$行动，未来能获得多少奖励。数学上表示为：

$$
V_{\pi}(s_t)=\mathbb{E}_\pi[R_t\mid s_t]
$$

展开期望计算：

$$
V_{\pi}(s)=\mathbb{E}_{\pi}\left[R_{t}+\gamma G_{t+1}|S_{t}=s\right]=\mathbb{E}_{\pi}\left[R_{t}+\gamma V_{\pi}(S_{t+1})\right|S_{t}=s]
$$

这个递归形式被成为**贝尔曼方程（Bellman equation）**，用于计算状态值函数

关于贝尔曼方程和相关推导，打算之后专门写一篇笔记记录一下

# 3.动作价值函数 Action-Value Function

动作价值函数$Q(s,a)$衡量在状态$s$下采取特定动作$a$后，按照策略$\pi$继续执行时的期望回报：

$$
Q_{\pi}(s,a)=\mathbb{E}_\pi[G_t|S_t=s,A_t=a]
$$

同样展开期望：

$$
Q_{\pi}(s,a)=\mathbb{E}_{\pi}\left[R_{t}+\gamma G_{t+1}|S_{t}=s,A_{t}=a\right]\\=\mathbb{E}_\pi\left[R_t+\gamma V_{\pi}(S_{t+1})|S_t=s,A_t=a\right]
$$

这表示在状态$s$采取动作$a$后，未来的累积回报的期望。

# 4.价值函数和动作价值函数的关系

状态值函数$V(s)$和动作价值函数$Q(s,a)$之间的关系为：

$$
V_{\pi}(s)=\mathbb{E}_\pi[Q_{\pi}(s,A)|S_t=s]
$$

也就是说，在状态$s$处的状态值等于所有可能动作的动作值的加权平均（权重是策略$\pi(a|s)$）。

展开：

$$
V_{\pi}(s)=\sum_a\pi(a|s)Q_{\pi}(s,a)
$$

# 5.优势函数 Advantage Function

优势函数$A(s,a)$衡量某个动作$a$在状态$s$处比平均值（即状态值）好多少：

$$
A_{\pi}(s,a)=Q_{\pi}(s,a)-V_{\pi}(s)
$$

这表示：

- 若$A_{\pi}(s,a)>0$，说明这个动作$a$比策略的平均表现更好，应该更倾向于选择它；
- 若$A_{\pi}(s,a)<0$，说明这个动作$a$比策略的平均表现更差，应该减少选择它的概率。

由于$Q(s,a)$本身可以通过贝尔曼方程表示：

$$
Q_{\pi}(s,a)=\mathbb{E}_\pi[R_t+\gamma V_{\pi}(S_{t+1})|S_t=s,A_t=a]
$$

因此：

$$
A_{\pi}(s,a)=\mathbb{E}_\pi[R_t+\gamma V_{\pi}(S_{t+1})|S_t=s,A_t=a]-V_{\pi}(s)
$$

可以看到，优势函数描述了某个动作带来的即时奖励和未来价值增量与平均状态价值的差异。

# 6.优势函数在PPO训练中的作用

在基于策略梯度的强化学习算法（如PPO）中，目标是最大化以下优化目标：

$$
L_{\mathrm{policy}}=\mathbb{E}\left[\frac{\pi(a_t|s_t)}{\pi_{\mathrm{old}}(a_t|s_t)}A_{\pi}(s_t,a_t)\right]
$$

其中：

- $\frac{\pi(a_t|s_{t})}{\pi_{old}(a_{t}|s_{t})}$是重要性采样权重，体现新策略和旧策略的差异
- $A_{\pi}(s_{t},a_{t})$是优势函数，指示应该增加或减少某个动作的概率

优势函数的作用：

1. 如果$A_{\pi}(s_{t},a_{t})>0$，说明该动作比平均策略更好，应该增大新策略对该动作的概率（即优化方向）
2. 如果$A_{\pi}(s_{t},a_{t})<0$，说明该动作比平均策略更差，应该减少新策略对该动作的概率

> 如果值函数$V_{\pi}(s)$训练得很好，使得$V_{\pi}(s)\approx G_{t}$，那么优势值$A_{\pi}(s,a)$会变得接近0，这意味着梯度更新变小，策略更新变慢，这就是为什么PPO训练过程中值函数优化需要适度，不能过拟合
> 

挖个坑，以后单独写一篇PPO的笔记

# 7.关系总结

四者的关系可以总结如下：

$$
\begin{gathered}G_t=\sum_{k=0}^\infty\gamma^kR_{t+k} \\V_{\pi}(s)=\mathbb{E}_\pi[G_t|S_t=s] \\Q_{\pi}(s,a)=\mathbb{E}_\pi[R_t+\gamma V_{\pi}(S_{t+1})|S_t=s,A_t=a] \\A_{\pi}(s,a)=Q_{\pi}(s,a)-V_{\pi}(s)\end{gathered}
$$

以PPO的训练过程举例，它们的作用是：

- 回报$G_{t}$：是监督信号，指导智能体优化策略
- 值函数$V_{\pi}(s)$：是基准参考值，衡量当前状态的长期回报
- 动作值函数$Q_{\pi}(s,a)$：衡量选择某个动作后的回报
- 优势函数$A_{\pi}(s,a)$：是PPO的核心，告诉策略哪个动作比较好

# 8.直观理解

- 回报$G_{t}$是智能体真正获得的总收益
- 值函数$V_{\pi}(s)$是智能体预测的收益预期
- 动作值函数$Q_{\pi}(s,a)$是智能体对某个特定动作的收益估计
- 优势函数$A_{\pi}(s,a)$告诉智能体某个动作是否比平均水平更好