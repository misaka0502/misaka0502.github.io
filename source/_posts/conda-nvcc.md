---
title: Ubuntu服务器上在conda环境中安装nvcc并成功编译cuda扩展（无管理员权限）
date: 2026-01-14 19:35:27
tags:
    - Ubuntu
    - Conda
    - nvcc
    - cuda
    - debug
---
<script src="https://fastly.jsdelivr.net/gh/misaka0502/live2d-widget@V0.2/autoload.js"></script>
<!-- <script src="/live2d-widget/autoload.js"></script> -->

> 场景：在服务器上安装一个带CUDA扩展的Pyhton包（例如Pointnet++的pointnet2_ops）
问题：服务器GPU是RTX 3090，pytorch版本是2.4.1+cu118，但是系统上没有安装cuda 11.8，导致 `pip install -e .` 编译扩展失败。并且没有管理员权限，安装不了cuda 11.8.
目标：不触碰系统级CUDA，用conda在用户态装齐编译链，最终让`torch.utils.cpp_extension` 正常通过编译。
> 

我在服务器上要编译一个 PyTorch CUDA 扩展（典型的 `pointnet2_ops` 这种），机器显卡是 RTX 3090。按理说这类扩展编译并不复杂：`pip install -e .` 或 `python setup.py develop` 就能走完。但现实是：服务器上 CUDA 环境和我当前 PyTorch 的 CUDA 版本对不上，而且我**没有管理员权限**，不能装系统级 CUDA Toolkit。

## 0. 背景：为什么需要nvcc？

很多 PyTorch CUDA 扩展（比如 pointnet2_ops）会同时编译：

- `.cu`（CUDA 源码）：必须用 **nvcc** 编译
- `.cpp`（C++ 源码）：用 g++ 编译，但可能会 include CUDA 头文件（比如 `cuda_runtime_api.h`）

所以只要项目里有 `.cu` 或者 C++ 里 include 了 CUDA runtime 相关头文件，你就需要一个**完整可用的 CUDA Toolkit 头文件与编译工具链**。注意：

- **驱动 (nvidia driver)** 只负责运行时（能跑 GPU）
- **CUDA Toolkit** 才包含 `nvcc`、`cuda_runtime.h`、`cuda_runtime_api.h` 等**开发/编译**所需的文件

很多服务器“能跑 GPU”（`nvidia-smi` 正常）并不代表“能编译 CUDA 扩展”（`nvcc` 缺失或版本不对）。

## 1. 症状：编译时报 `cuda_runtime.h` / `cuda_runtime_api.h` 找不到

编译过程中会看到类似的报错（关键点是找不到CUDA runtime头文件）：

- `cc1plus: fatal error: cuda_runtime.h: No such file or directory`
- `fatal error: cuda_runtime_api.h: No such file or directory`

这类错误通常意味着：**编译器 include 路径里没有 CUDA Toolkit 的 include 目录**。

这时即使你机器上有 GPU、甚至 PyTorch 是 CUDA 版，也不保证编译能找到 CUDA 头文件。

## 2. 根因：服务器没有对应版本的nvcc/toolkit，且我没有管理员权限

很多集群环境会出现这样的情况;

- 系统只安装了driver（能运行）
- 没安装CUDA Toolkit或没安装你需要的版本
- 或者装了，但你没有权限修改系统路径/安装新版本

而CUDA扩展编译时，PyTorch的`torch.utils.cpp_extension`会调用nvcc（路径通常来自`CUDA_HOME`或`PATH`），并在编译命令里假设能include到`cuda_runtime*.h` 。

所以根因可以概括为一句话：

> 运行时环境有GPU，但缺少（或找不到）用于编译的CUDA Toolkit（nvcc+headers）。
> 

## 3. 解决策略：不用sudo，用conda在环境中装一套nvcc=11.8+toolkit

没有管理员权限时，最稳的思路就是：

> 直接在conda环境中安装CUDA Toolkit（含nvcc）并让编译系统优先使用这套工具链。
> 

我用的命令是：

```bash
conda install -c nvidia cuda-nvcc=11.8 cuda-toolkit=11.8
```

解释一下：

- `cuda-nvcc=11.8`: 提供nvcc编译器
- `cuda-toolkit=11.8`: 提供headers、libs等开发文件（非常关键）
- `-c nvidia`: nvidia官方conda channel，包更完整、版本更对齐

装完后，nvcc会出现在`$CONDA_PREFIX/bin/nvcc` 。

## 4. 配置：让编译系统找到conda里的CUDA（CUDA_HOME/PATH/LD_LIBRARY_PATH）

装完不代表自动生效。要让`torch.utils.cpp_extension`以及ninja/g++/nvcc一整条链路都用conda这套CUDA，需要配置环境变量;

```bash
export CUDA_HOME=$CONDA_PREFIX
export PATH=$CUDA_HOME/bin:$PATH
export LD_LIBRARY_PATH=$CUDA_HOME/lib:$LD_LIBRARY_PATH
```

这三行的作用：

- `CUDA_HOME=$CONDA_PREFIX`: 告诉构建系统”CUDA Toolkit 根目录在conda env里”
- `PATH=$CUDA_HOME/bin:$PATH`: 确保调用的是conda里的nvcc
- `LD_LIBRARY_PATH=…` : 运行/链接阶段更容易找到conda里的CUDA动态库（有时编译链接也会用到）

然后检查nvcc是否生效：

```bash
nvcc -V
```

看到版本为11.8基本就可以了。

## 5. 额外验证：CUDA头文件到底在不在codna环境里？

出现`cuda_runtime.h`找不到时，有一个最有效的sanity check: 

```bash
find $CONDA_PREFIX -name cuda_runtime.h | head
find $CONDA_PREFIX -name cuda_runtime_api.h | head
```

如果能在codna env下面找到它们（比如在 `.../targets/x86_64-linux/include/`），说明工具链是齐的；接下来只剩“编译时include路径是否正确引用到了它们”。

## 6. 为什么这套方法能生效？

核心原理就是：

> 把系统级CUDA Toolkit替换成conda env内的一套CUDA Toolkit
> 

PyTorch CUDA扩展编译时依赖三个东西;

1. nvcc (编译`.cu`)
2. CUDA headers (`cuda_runtime*.h`等)
3. CUDA libs (链接阶段)

conda的`cuda-toolkit`+`cuda-nvcc`正好把这三件事都塞进了`$CONDA_PREFIX` ，而`CUDA_HOME/PATH/LD_LIBRARY_PATH` 又把构建系统的“默认搜索路径”指向了conda env，因此构建系统不需要管理员权限，也不用系统安装CUAD Toolkit，就能完整编译扩展。

## 7. 最终结果：pointnet2_ops成功编译并安装

当codna里的nvcc/toolkit生效、路径配置正确之后，重新执行安装：

```bash
pip install -e .
```

编译过程顺利完成，CUDA扩展成功生成并安装到环境中。

## 8. 经验总结

遇到“能跑CUDA、不能编译CUDA扩展”时，优先按照这条路径排查：

- 第一类问题：没有nvcc/nvcc版本不对
    - `nvcc -V`看不到版本或版本不匹配
    - 用conda安装：`cuda-nvcc` + `cuda-toolkit`
- 第二类问题：头文件找不到
    - `cuda_runtime.h`/`cuda_runtime_api.h` not found
    - `find $CONDA_PREFIX -name cuda_runtime.h` 验证文件确实存在
    - 然后检查`CUDA_HOME`/include path 是否指向conda env
- 第三类问题：链接库找不到
    - 可能需要`LD_LIBRARY_PATH`或者conda env的lib路径更完整

一句话：驱动≠工具链。集群给你driver不等于给你nvcc；没有sudo也不等于不能编译，只要你把工具链装进conda并把路径指过去。