---
title: Flow matching训练时训练损失不降低的问题
date: 2025-09-11 10:06:26
tags:
    - algorithm
    - flow matching
    - diffusion
    - torch
---
<script src="https://fastly.jsdelivr.net/gh/misaka0502/live2d-widget@V0.2/autoload.js"></script>
<!-- <script src="/live2d-widget/autoload.js"></script> -->

# Flow matching训练时训练损失不降低

今天尝试复现flow matching训练一个机器人操作任务，但是发现训练损失完全没有下降的趋势，排查之后发现问题在于网络初始化的时机不对。

在我的流匹配模型中(MLP based)，一开始的网络初始化逻辑是这样的:

```python
# In FlowMatchingMLP

def __init__(...):
    # ...
    self.time_mlp = nn.Sequential(
        nn.Linear(time_embedding_dim, time_embedding_dim),
        nn.SiLU(),
        nn.Linear(time_embedding_dim, time_embedding_dim),
    )
    # These will be initialized dynamically based on input shape
    self.input_proj = None
    self.cond_proj = None
    # ...
    self._layers_initialized = False

def forward(...):
    # ...
    # Initialize layers if needed
    self._initialize_layers(T * D, x.device)
    # ...
```

即主要部分的初始化是在第一次调用forward时执行的，这里存在一个严重的问题。

PyTorch 的优化器 (如 Adam) 在被创建时，会接收模型的参数列表 (model.parameters())。这个过程**通常发生在训练循环开始之前**。

在之前的实现中，当优化器被创建时，`self.input_proj`, `self.net`, `self.output_proj` 等都还是 None。它们并不是 `nn.Module`，因此它们的参数**不会被注册到优化器中**。

结果就是，即使在 forward 过程中这些层被创建了，优化器也“看”不到它们。因此，在 `optimizer.step()` 调用时，这些层的梯度虽然可能被计算了，但**它们的权重永远不会被更新**。模型的核心部分实际上是“冻结”的，只有 `time_mlp` 的参数在学习，这显然不足以让模型学会任何有意义的东西，导致损失完全不下降。

那么解决方案也很简单：**将所有层的定义都移到 init 方法中。**

进行如上修改后训练损失就能正常降低了。