---
title: 广义优势估计（Generalized Advantage Estimation，GAE）
date: 2025-04-23 23:57:01
tags:
    - 强化学习
mathjax: true
banner_img: /images/bg/hina_kisaki.jpg
---
<script src="https://fastly.jsdelivr.net/gh/misaka0502/live2d-widget@V0.2/autoload.js"></script>
<!-- <script src="/live2d-widget/autoload.js"></script> -->

**Warning：这是一篇GPT-based笔记**

> 在强化学习中，广义优势估计（Generalized Advantage Estimation，GAE）是一种对优势函数进行平滑估计的方法，通过对不同步长的时序差分残差（TD residual）按指数加权累积，以在**偏差-方差权衡**上取得最佳效果。GAE引入了参数$\lambda$控制权重，当$\lambda →0$时退化为单步TD(0)，可获得低方差、高偏差的估计；当$\lambda→1$时退化为蒙特卡洛回报，具有低偏差、高方差的特点。在实践中，选择中间值的$\lambda$常常能显著提高策略梯度算法（如PPO）的样本效率和稳定性。
> 

# 一、背景：优势函数与偏差-方差权衡

## 1.优势函数（Advantage Function）

- 优势函数定义为
  
    $$
    A^\pi(s,a)=Q^\pi(s,a)-V^\pi(s),
    $$
    
    它衡量在状态$s$下执行动作$a$相较于策略平均水平的收益增益。
    
- 在策略梯度中，使用优势函数可以去除与状态无关的常数基线V，从而降低方差，但仍保持无偏。

## 2.偏差-方差权衡

- **蒙特卡洛回报**完整依赖未来真实奖励，**无偏**但因为来轨迹随机性导致**高方差**
    - 蒙特卡洛方法高方差、低偏差
    - 由于完全依赖真是回报，不使用估计值，所以是**无偏估计器**，只要采样足够，期望就等于真实的$V^{\pi}(s)$；但完整的回报$G_t$受到整个未来路径上所有的随机性影响（包括未来的动作选择、转移、奖励），所以路径之间差异很大，导致估计**波动很大**
- **时序差分（TD）方法**仅使用一步奖励加上价值估计，**方差低**但因依赖估计值引入**偏差**
    - 时序差分方法低方差、高偏差
    - 因为TD方法只用一步奖励和一个状态的估计，不用考虑整个未来轨迹，所以**波动较小**，且TD以“逐步更新”的方法稳定收敛；TD的更新依赖当前的$V(s')$估计，如果这个估计本身有误，就会将误差“传播”到当前状态中，引入偏差，所以TD是**有偏估计器**，但这个偏差会随时间下降
- GAE通过$\lambda$-加权方案，将两者的优缺点平滑折中

# 二、GAE的定义与公式

## 1.TD残差

对任意时间步$t$，定义一阶TD残差（有些地方也称作TD误差，是一个概念）：

$$
\delta_t=r_{t+1}+\gamma V(s_{t+1})-V(s_t).
$$

该残差是标准TD(0)更新中的核心量。

## 2.指数加权累积

GAE将不同步长的TD残差按指数权重累积，得到对任意$t$的广义优势估计：

$$
\hat{A}_t^{\mathrm{GAE}(\gamma,\lambda)}=\sum_{l=0}^\infty(\gamma\lambda)^l\delta_{t+l}.
$$

其中$\lambda \in [0,1]$控制对更远残差的衰减。

## 3.特殊情况

- 当$\lambda=0$：
    - $\hat{A}_{t}=\delta_{t}$，退化为TD(0)，方差最低但是偏差最高。
- 当$\lambda=1$：
    - $\hat{A}_t=\sum_{l=0}^\infty\gamma^l\delta_{t+l}=G_t-V(s_t)$，退化为蒙特卡洛优势估计，偏差最小但是方差最大。

# 三、GAE的推导与直观理解

## 1.从$\lambda$-return角度：

$\lambda$-return定义为对不同步长n-step目标按$(1-\lambda)\lambda ^{n-1}$加权，GAE则在此基础上提出基线$V(s_{t})$得到优势估计。

## 2.从偏差-方差折中：

- $\lambda$越接近0，更依赖单步估计，减少方差但累计偏差；
- $\lambda$越接近1，更依赖完整回报，减少偏差但增加方差；

## 3.动态规划视角：

将多步TD误差递归地累积，视作对未来价值估计误差的“指数衰减”补偿

# 四、算法实现要点

- 在PPOBuffer等回放缓存中，GAE计算通常按倒序进行：
  
    ```python
    adv = 0
    for t in reversed(range(T)):
        delta = rewards[t] + gamma * V[t+1] - V[t]
        adv = delta + gamma * lambda_ * adv
        advantages[t] = adv
    ```
    
    最终将$\hat A_{t}$用于策略梯度更新。
    
- 同时可计算回报目标$\hat R_{t}=\hat A_{t}+V(s_t)$，用于价值网络的训练

# 五、应用与实验效果

- PPO、TRPO等主流策略梯度算法广泛采用GAE，当$\lambda ≈0.95$时常能达到最优效果。
- 在高维连续控制任务上，相比固定n-step或纯蒙特卡洛，GAE显著提升了样本效率与训练稳定性。
- 多篇后续工作也证实了GAE在复杂策略和大规模分布式训练中的优越性。

# 六、小结

广义优势估计（GAE）通过对TD残差进行指数加权累积，引入$\lambda$控制偏差-方差权衡，兼具低方差与低偏差的优势。在实践中，GAE已成为策略梯度算法（尤其是PPO/TRPO）的标配组件，有效提升了收敛速度和策略性能，是现代深度强化学习中不可或缺的关键技术之一。