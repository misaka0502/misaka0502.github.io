---
title: DDPM(Denoising Diffusion Probabilistic Models)论文阅读笔记
date: 2024-12-04 23:23:01
tags: 
    - 深度学习
    - Diffusion Model
mathjax: true
banner_img: /images/bg/emt1.png
---
<script src="https://fastly.jsdelivr.net/gh/misaka0502/live2d-widget@V0.2/autoload.js"></script>
<!-- <script src="/live2d-widget/autoload.js"></script> -->

# DDPM(Denoising Diffusion Probabilistic Models)论文阅读笔记

> 论文：[Denoising Diffusion Probabilistic Models](https://arxiv.org/abs/2006.11239)
> 

# 前言

最近在研究Diffusion Policy，跟DDPM关系非常密切，于是读了下DDPM的文章，写篇笔记，顺便挖个坑，记录一些生成模型的研究。DDPM（以下简称扩散模型）不仅在图像生成方面有着非常广泛的应用，而且现在在机器人学习也有研究前景。在读了扩散模型的论文和一些解读的博客后，发掘其中有许多细节需要仔细探讨。（实际还有很多细节还没理解🤡）

# 什么是Diffusion Model（扩散模型）？

首先需要理清一点，借用[生成扩散模型漫谈（一）：DDPM = 拆楼 + 建楼](https://spaces.ac.cn/archives/9119/comment-page-1)一文所述，传统的扩散模型实际上是一类模型，涉及到能量模型（Energy-based Models）、得分匹配（Score Matching）、朗之万方程（Langevin Equations）等等，DDPM也用了“扩散模型”这一名称，但实际上除了采样过程的形式有一定相似处外，DDPM与传统基于朗之万方程采样的扩散模型可以说完全不一样，传统的扩散模型的能量模型、得分匹配、朗之万方程等概念，其实跟DDPM及其后续变体都没什么关系。

![image.png](/images/ddpm_1.png)

扩散模型的核心是两个过程/概念：**前向过程（加噪过程、扩散过程）和反向过程（去噪过程）**。直白理解，前向过程就是**逐步**地往一张完好的图像里添加噪声，让其逐渐靠近高斯噪声，整个过程可以表示为：

$$
x=x_0 \to x_1 \to x_2 \to ··· \to x_{T-1} \to x_T=z \tag{1}
$$

现在在前向过程中，我们知道每一步加噪的过程，即$x_{t-1} \to x_{t}$，如果我们能通过某种方式找到其对应的反向过程，即$x_t \to x_{t-1}$，将这种变换关系记作$x_{t-1}=\mu(x_t)$，反复地执行$x_{T-1}=\mu(x_T)、x_{T-2}=\mu(x_{T-1}) ···$，是不是就能**逐步**地从$x_T$还原回原始的图像$x_0$？

以上就是扩散模型最核心的思想。

这里首先要补充一个重要特性：

## 重参数（R**eparameterization**）

重参数技巧在很多工作中有所引用。如果我们要从某个分布中随机采样 (高斯分布) 一个样本，**这个过程是无法反传梯度的**。而这个通过高斯噪声采样得到$x_t$的过程在 diffusion 中到处都是，因此我们需要通过重参数技巧来使得他可微。

最通常的做法是把随机性通过一个独立的随机变量 (ϵ) 引导过去。即如果要从高斯分布$z\thicksim\mathcal{N}(z;\mu_\theta,\sigma^2_\theta \mathbf{I})$采样一个$z$，我们可以写成：

$$
z=\mu_\theta+\sigma_\theta \odot \epsilon,\quad \epsilon \thicksim\mathcal{N}(0,\mathbf{I})
$$

上式的$z$依旧是有随机性的，且满足均值为$\mu_\theta$，方差为$\sigma^2_\theta$的高斯分布。这里的$\mu_\theta$，$\sigma^2_\theta$可以是由参数$\theta$的神经网络推断得到的。整个采样过程依旧梯度可导，随机性被转嫁到了$\epsilon$上。

# 前向过程 （**Forward Diffusion Process**）

首先声明几个符号的关系：令$\alpha_t=1-\beta_t,\bar{\alpha_t}=\prod^{t}_{i=1}\alpha_i$

DDPM将前向过程建模为

$$
x_t=\sqrt{\alpha_t}x_{t-1}+\sqrt{\beta_t}\epsilon_t,\quad \epsilon_t \thicksim \mathcal{N}(0, 1)  \tag{2}
$$

这个公式就是一步步给图像添加噪声的公式。

$\beta_t$通常很接近0，可以理解为加噪过程中对原图像的损坏程度，噪声$\epsilon_t$的引入代表着对原始信号的一种破坏，引用论文的原话：

> The forward process variances $\beta_t$ can be learned by reparameterization or held constant as hyperparameters, and expressiveness of the reverse process is ensured in part by the choice of Gaussian conditionals in  $p_{\theta}(x_{t-1}|x_t)$， because both processes have the same functional form when  $\beta_t$  are small.
正向过程方差 $\beta_t$ 可以通过重新参数化来学习，或者作为超参数保持不变，而逆过程的表达能力部分由 $p_{\theta}(x_{t-1}|x_t)$中高斯条件的选择来保证，**因为当 $\beta_t$ 很小时，两个过程具有相同的函数形式。**
这里的 $p_{\theta}(x_{t-1}|x_t)$后面会讲到
> 

这个前向过程也可以用

$$
q(x_t|x_{t-1})=\mathcal{N}(x_t;\sqrt{\alpha_t}x_{t-1},(1-\alpha_t)\mathbf{I}) \tag{3}
$$

来表示，也被称为“近似后验”（approximate posterior），整个前向过程符合马尔可夫性质，是一个马尔科夫链。

图示：

![image.png](/images/ddpm_2.png)

反复执行加噪步骤，我们可以推导：

![image.png](/images/ddpm_3.png)

这是前向过程中的关键性质。也可以表示为：

$$
q(x_t|x_0)=\mathcal{N}(x_t;\sqrt{\bar{\alpha_t}}x_0,(1-\bar{\alpha_t})\mathbf{I}) \tag{4}
$$

这被称为**Close form。**

从这个形式，我们可以得出两个结论：

- 任意时刻的$x_t$可以由$x_0$和$\beta$表示，这就为计算$x_t$提供了极大的便利。
- 通过选择合适的$\alpha_t$（论文里是人为设置$\beta_t$），当时间步$T\to \infty$时，$\bar{\alpha_t} \to 0$，$x_T$实际上会被转换为高斯噪声。

以上就是前向过程，不涉及到训练。

# 反向过程 （**Reverse Diffusion Process**）

如果把前向过程视作“加噪”过程，那么反向过程就是“去噪”过程，即$x_{t-1} \to x_t$的过程。

![image.png](/images/ddpm_4.png)

在前向过程中，$q(x_t|x_{t-1})$是已知的，如果能求解目标分布$q(x_{t-1}|x_{t})$，这个问题救迎刃而解了。然而，我们无法简单推断$q(x_{t-1}|x_{t})$。因此换个思路，我们知道神经网络是通用函数近似器，可以利用神经网络去预测一个反向的过程$p_\theta(x_{t-1}|x_{t})$，但是要怎么预测、怎么训练？这是反向过程的核心问题。

已经有文献证明，如果$q(x_t|x_{t-1})$满足高斯分布且**$\beta_t$**足够小，$q(x_{t-1}|x_{t})$仍然是一个高斯分布。因此我们可以假定$p_\theta(x_{t-1}|x_{t})$也是一个高斯分布：

$$
p_\theta(x_{t-1}|x_t)=\mathcal{N}(x_{t-1};\mu_\theta(x_t,t),\Sigma_\theta(x_t,t)) \tag{5}
$$

前面说到，反向过程是训练一个网络$p_\theta(x_{t-1}|x_{t})$去预测$q(x_{t-1}|x_{t})$，但是$q(x_{t-1}|x_{t})$我们无法直接得到，那么该怎么训练？这里引入了一个很巧妙的方法，那就是我们不去找$q(x_{t-1}|x_{t})$，而是找$q(x_{t-1}|x_{t},x_0)$，把条件分布变成联合分布。由于前向过程符合马尔可夫性质，所以这两个东西实际是等价的，但是后者是可以通过贝叶斯公式推导得到的，推导过程如下：

$$
q(x_{t-1}|x_t,x_0)=\frac{q(x_t,x_0,x_{t-1})}{q(x_t,x_0)} \\
=\frac{q(x_0)q(x_{t-1}|x_0)q(x_t|x_{t-1},x_0)}{q(x_0)q(x_t|x_0)} \\
=q(x_t|x_{t-1},x_0)\frac{q(x_{t-1}|x_0)}{q(x_t|x_0)} \tag{6}
$$

至此，将反向过程全部变回了正向过程，而正向过程我们是知道的，这里的每一项都可以求。

> 这里有一个容易绕晕的点（我自己一开始也绕晕了），既然反向过程都已经表示出来了，为什么不能直接用呢？因为这里讨论的是训练过程，$x_0$我们是知道的，而推理时$x_0$是不知道的。
> 

式（7）中的各项也都是符合高斯分布的，如下：

![image.png](/images/ddpm_5.png)

因此，式（7）可以继续往下推导（推导过程省略了，可以通过高斯分布的表达式推，直接给出结果）：

$$
q(x_{t-1}|x_t,x_0)=\mathcal{N}(x_{t-1},\frac{\sqrt{\alpha_t}(1-\bar{\alpha}_{t-1})x_t+\sqrt{\bar{\alpha}_{t-1}}(1-\alpha_t)x_0}{1-\bar{\alpha}_t},\frac{(1-\alpha_t)(1-\bar{\alpha}_{t-1})}{1-\bar{\alpha}_t}\mathbf{I}) \tag{7}
$$

在这里，我们令$\tilde{\mu}_t(x_t,x_0)=\frac{\sqrt{\alpha_t}(1-\bar{\alpha}_{t-1})x_t+\sqrt{\bar{\alpha}_{t-1}}(1-\alpha_t)x_0}{1-\bar{\alpha}_t}$，$\tilde{\beta}_t=\frac{(1-\alpha_t)(1-\bar{\alpha}_{t-1})}{1-\bar{\alpha}_t}$。

由于$\beta_1,\beta_2···\beta_t$可以是训练时人为设置的一组超参数，所以这里$\tilde{\beta}_t$可以视为一个常数。因此，训练的关键其实落在了如何预测$\tilde{\mu}_t(x_t,x_0)$。

由Close-Form可以求出：

$$
x_{0}=\frac{1}{\sqrt{\bar{\alpha}_t}}(x_t-\sqrt{1-\bar{\alpha}_t}\epsilon_t) \tag{8}
$$

将式（8）代入式（7）中，可以将$\tilde{\mu}_t(x_t,x_0)$化简为：

$$
\tilde{\mu}_t(x_t,x_0)=\frac{1}{\sqrt{\alpha_t}}(x_t-\frac{1-\alpha_t}{\sqrt{1-\bar{\alpha}_t}}\epsilon_\theta(x_t,t)) \tag{9}
$$

这个式子中，也只有$\epsilon_t$是未知的，因此，我们实际上是需要神经网络去预测一个“**噪声**”。

在训练好网络后，我们就可以通过前文提到的重参数从$x_T$去一步步**采样**得到$x_0$：

$$
x_{t-1}=\frac{1}{\sqrt{\alpha_t}}(x_t-\frac{1-\alpha_t}{\sqrt{1-\bar{\alpha}_t}}\epsilon_\theta(x_t,t))+\sigma_tz \tag{10}
$$

为了进行随机采样，这里的最后有一个高斯项。

以上就是反向过程，核心就是设计一个神经网络去预测分布$q(x_{t-1}|x_t,x_0)$的均值$\mu_\theta(x_t,t)$，而预测均值实际上是通过预测噪声$\epsilon_\theta(x_t,t)$实现的。

总结反向过程：

1. 每个时间步通过$x_t$和$t$来预测高斯噪声$\epsilon_\theta(x_t,t)$，随后根据式（9）得到均值$\mu_\theta(x_t,t)$
2. 得到方差$\Sigma_\theta(x_t,t)$，在DDPM中使用untrained$\Sigma_\theta(x_t,t)=\tilde{\beta}_t$，且认为$\tilde{\beta}_t=\beta_t$和$\tilde{\beta}_t=\frac{1-\overline{\alpha}_{t-1}}{1-\overline{\alpha}_t}·\beta_t$结果近似
3. 根据式（5）得到$q(x_{t-1}|x_t)$，利用重参数得到$x_{t-1}$

# 训练过程

## 优化目标——最大似然估计

这里其实我没看懂，简单记录一些式子，等以后看懂了继续补充。

论文原话：Training is performed by optimizing the usual variational bound on negative log likelihood.

$$
\mathbb{E}\left[-\log p_\theta(\mathbf{x}_0)\right]\leq\mathbb{E}_q\left[-\log\frac{p_\theta(\mathbf{x}_{0:T})}{q(\mathbf{x}_{1:T}|\mathbf{x}_0)}\right] \\
=\mathbb{E}_q\left[-\log p(\mathbf{x}_T)-\sum_{t\geq1}\log\frac{p_\theta(\mathbf{x}_{t-1}|\mathbf{x}_t)}{q(\mathbf{x}_t|\mathbf{x}_{t-1})}\right]=:L \tag{11}
$$

下面分析一下。实际上这是在优化$x_0 \thicksim q(x_0)$下的$p_\theta(x_0)$交叉熵：

$$
\mathcal{L}=\Bbb{E}_{q(x_0)}[-logp_\theta(x_0)] \tag{12}
$$

可以使用变分下界（VLB）来优化负对数似然。由于KL散度非负，可得到：

$$
\begin{aligned}-\log p_\theta(x_0) & \leq-\log p_\theta(x_0)+D_{KL}(q(x_{1:T}|x_0)||p_\theta(x_{1:T}|x_0)) \\ & =-\log p_\theta(x_0)+\mathbb{E}_{q(x_{1:T}|x_0)}\left[\log\frac{q(x_{1:T}|x_0)}{p_\theta(x_{0:T})/p_\theta(x_0)}\right];\quad\mathrm{~where~}\quad p_\theta(x_{1:T}|x_0)=\frac{p_\theta(x_{0:T})}{p_\theta(x_0)} \\ & =-\log p_\theta(x_0)+\mathbb{E}_{q(x_{1:T}|x_0)}\left[\log\frac{q(x_{1:T}|x_0)}{p_\theta(x_{0:T})}+\underbrace{\log p_\theta(x_0)}_{\text{与}q\text{无关}}\right] \\ & =\mathbb{E}_{q(x_{1:T}|x_0)}\left[\log\frac{q(x_{1:T}|x_0)}{p_\theta(x_{0:T})}\right].  \end{aligned} \tag{13}
$$

这就是ELBO（证据下界/变分下界）。对左右取期望$\Bbb{E}_{q(x_0)}$，利用重积分中的Fubini定理，可以得到：

$$
\mathcal{L}_{VLB}=\underbrace{\mathbb{E}_{q(x_0)}\left(\mathbb{E}_{q(x_{1:T}|x_0)}\left[\log\frac{q(x_{1:T}|x_0)}{p_\theta(x_{0:T})}\right]\right)=\mathbb{E}_{q(x_{0:T})}\left[\log\frac{q(x_{1:T}|x_0)}{p_\theta(x_{0:T})}\right]}_{Fubini\text{定理}} \\
\geq\mathbb{E}_{q(x_0)}[-\log p_\theta(x_0)]. \tag{14}
$$

能够最小化$\mathcal{L}_{VLB}$即可最小化我们的目标损失(11)。

进一步对$\mathcal{L}_{VLB}$推导，可以得到熵与多个KL散度的累加，具体可见文献【】中的推导。这里直接给出结果：

$$
\mathbb{E}_q\left[\underbrace{D_{\mathrm{KL}}(q(\mathbf{x}_T|\mathbf{x}_0)\parallel p(\mathbf{x}_T))}_{L_T}+\sum_{t>1}\underbrace{D_{\mathrm{KL}}(q(\mathbf{x}_{t-1}|\mathbf{x}_t,\mathbf{x}_0)\parallel p_\theta(\mathbf{x}_{t-1}|\mathbf{x}_t))}_{L_{t-1}}\underbrace{-\log p_\theta(\mathbf{x}_0|\mathbf{x}_1)}_{L_0}\right] \tag{15}
$$

由于前向q没有可学习的参数，而$x_T$则是纯高斯噪声，$L_T$可以当作常量忽略。而$L_t$则可以看作拉近2个高斯分布

$q(x_{t-1}|x_t,x_0)=\mathcal{N}(x_{t-1;}\tilde{\mu}(x_t,x_0),\tilde{\beta}_{t}\mathbf{I})$和$p_\theta(x_{t-1}|x_t)=\mathcal{N}(x_{t-1},\mu_\theta(x_t,t),\Sigma_\theta)$，根据多元高斯分布的KL散度求解：

$$
L_{t-1}=\mathbb{E}_{q}\left[\frac{1}{2 \sigma_{t}^{2}}\left\|\tilde{\boldsymbol{\mu}}_{t}\left(\mathbf{x}_{t}, \mathbf{x}_{0}\right)-\boldsymbol{\mu}_{\theta}\left(\mathbf{x}_{t}, t\right)\right\|^{2}\right]+C \tag{16}
$$

DDPM在训练时使用的是以下变体：

$$
L_{\mathrm{simple}}(\theta):=\mathbb{E}_{t,\mathbf{x}_{0},\boldsymbol{\epsilon}}\left[\left\|\epsilon-\epsilon_{\theta}(\sqrt{\bar{\alpha}_{t}}\mathbf{x}_{0}+\sqrt{1-\bar{\alpha}_{t}}\boldsymbol{\epsilon},t)\right\|^{2}\right] \tag{17}
$$

总结训练过程：

1. 获取输入$x_0$，从$1...T$随机采样一个t
2. 从标准高斯分布采样一个噪声$\epsilon_t\thicksim \cal{N}(0,\mathbf{I})$
3. 最小化$||\epsilon_t-\epsilon_\theta(\sqrt{\bar{\alpha}}_tx_0+\sqrt{1-\bar{\alpha}_t}\epsilon_t,t)||$

![image.png](/images/ddpm_6.png)

# 重要公式总结

1. forward（close-form）：
   
    $$
    x_t=\sqrt{\bar{\alpha}_t}x_0+\sqrt{1-\bar{\alpha}_t}\epsilon
    $$
    
2. optimization：
   
    $$
    L_{\mathrm{simple}}(\theta):=\mathbb{E}_{t,\mathbf{x}_{0},\boldsymbol{\epsilon}}\left[\left\|\epsilon-\epsilon_{\theta}(\sqrt{\bar{\alpha}_{t}}\mathbf{x}_{0}+\sqrt{1-\bar{\alpha}_{t}}\boldsymbol{\epsilon},t)\right\|^{2}\right]
    $$
    
3. reverse：
   
    $$
    q(x_{t-1}|x_t,x_0)=\mathcal{N}(x_{t-1},\frac{\sqrt{\alpha_t}(1-\bar{\alpha}_{t-1})x_t+\sqrt{\bar{\alpha}_{t-1}}(1-\alpha_t)x_0}{1-\bar{\alpha}_t},\frac{(1-\alpha_t)(1-\bar{\alpha}_{t-1})}{1-\bar{\alpha}_t}\mathbf{I})
    $$
    
4. training：
   
    $$
    \text{argmin}_{\theta} \frac{1}{2\sigma_q^2(t)} \langle \mu_\theta - \mu_q, \frac{1}{2} \rangle
    $$
    
    $$
    \text{argmin}_{\theta} \frac{1}{2\sigma_q^2(t)} \langle (1 - \alpha_t), \alpha_t \rangle \langle e_t, \epsilon_\theta(x_t, t) \rangle
    $$
    
5. sampling：
   
    $$
    x_{t-1}=\frac{1}{\sqrt{\alpha_t}}(x_t-\frac{1-\alpha_t}{\sqrt{1-\bar{\alpha}_t}}\epsilon_\theta(x_t,t))+\sigma_tz
    $$
    

DDPM论文提供的训练/测试（采样）伪代码：

![image.png](/images/ddpm_7.png)

# 其他细节

## 降低方差

原则上来说，优化损失函数（17）就可以完成DDPM的训练，但它在实践中可能有方差过大的风险，从而导致收敛过慢等问题。式（17）中实际包含了4个需要采样的随机变量：

1. 从所有训练样本中次采样一个$x_0$
2. 从正态分布$\cal{N}(0,\mathbf{I})$中采样$\epsilon_t$和$\epsilon_{t-1}$（两个不同的采样结果）
3. 从$1 \thicksim T$中采样一个$t$

要采样的随机变量越多，就越难对损失函数做准确的估计，反过来说就是每次对损失函数进行估计的波动（方差）太大了。实际上我们可以通过几个积分技巧将$\epsilon_t$和$\epsilon_{t-1}$合并成单个正态随机变量，从而缓解方差过大的问题。这一点就不详细记录了，因为我没看懂🥲

## 递归生成（采样）

训练完之后，我们就可以从一个随机噪声$x_T \thicksim \cal{N}(0,\mathbf{I})$出发执行$T$步采样公式（10）来进行生成。

> 一般来说，我们可以让$\sigma_t=\beta_t$，即正向和反向的方差保持同步。这个采样过程跟传统的扩散模型的朗之万采样不一样的地方在于：DDPM的采样每次都从一个随机噪声书法，需要重复迭代$T$步来得到一个样本输出；朗之万采样是从任意一个点出发，反符迭代无限步，理论上这个迭代无限步的过程中，就把所有数据样本都生成过了。所以两者除了形式相似之外，实质上是两个截然不同过的模型。
> 

![image.png](/images/ddpm_8.png)

## TODO

现在文章里用的符号有点乱，找个时间统一一下。

# 参考文献

[Denoising Diffusion Probabilistic Models](https://arxiv.org/abs/2006.11239)

[生成扩散模型漫谈（一）：DDPM = 拆楼 + 建楼](https://spaces.ac.cn/archives/9119)

[【大白话01】一文理清 Diffusion Model 扩散模型 | 原理图解+公式推导](https://www.bilibili.com/video/BV1xih7ecEMb/?spm_id_from=333.337.search-card.all.click&vd_source=418a13a1dcf4b812b80b357e2cc44de8)

[由浅入深了解Diffusion Model](https://zhuanlan.zhihu.com/p/525106459)