---
title: ninja / g++ / nvcc 到底谁是谁：一次把“编译这件事”的角色分清楚
date: 2026-01-14 19:39:49
tags:
    - Compile
    - Linux
    - CUDA
---
<script src="https://fastly.jsdelivr.net/gh/misaka0502/live2d-widget@V0.2/autoload.js"></script>
<!-- <script src="/live2d-widget/autoload.js"></script> -->

今天在服务器上编译pointnet2_ops时遇到了一些问题，并且报错的输出中包含以下信息:

- `ninja: build stopped`
- `c++ ...`（其实就是 g++）
- `/path/to/nvcc ...`
- 以及一堆 `-I` `-L` `-gencode` `-std=c++17`

于是去研究了一下这些工具在编译中的角色，特别是ninja / g++ / nvcc这三者。

## 1. 总览：三者的分工

| 工具 | 本质角色 | 负责什么 | 你会在什么时候看到它 |
| --- | --- | --- | --- |
| **g++**（或 clang++） | C/C++ 编译器 + 链接器前端 | 编译 `.cpp` → `.o`，并参与链接生成 `.so/.exe` | 任何 C++ 扩展、PyTorch extension、CMake 项目 |
| **nvcc** | CUDA 编译驱动（driver） | 处理 `.cu`，把 host 代码交给 g++/clang，把 device 代码交给 PTX/SASS 编译器，再组织链接 | 只要有 CUDA `.cu` 文件、`CUDAExtension` |
| **ninja** | 构建系统（build executor） | 按 build.ninja 的规则并行执行命令（调用 g++/nvcc 等） | CMake 或 PyTorch cpp_extension 自动生成 ninja 文件时 |

一句话记忆;

- g++/clang++: 真正把源码变成机器能跑的东西
- nvcc: CUDA场景里的编译总指挥，自己不一定亲自干活，但协调host/device两条路线
- ninja: 工地调度系统，负责谁先干、谁并行干、失败就停

## 2. g++：经典C++编译器（同时也是链接器入口）

### g++做什么？

对C++来说，最标准的一条流水线就是：

1. 预处理：展开`#include` 、宏
2. 编译：变成汇编
3. 汇编：变成目标文件`.o`
4. 链接：把一堆`.o` 和库拼成`.so` 或可执行文件

### g++关心哪些路径？

- 头文件：`I...`（以及环境变量 `CPATH` 等）
- 链接库：`L... -lxxx`（以及 `LIBRARY_PATH`）
- 运行时库：生成完程序以后由系统 loader 找（`LD_LIBRARY_PATH`）

## 3. nvcc：CUDA编译“驱动”，不是单一编译器

很多人误会nvcc是“CUDA的g++”。更准确的说法是：

> **nvcc 是一个“编译驱动程序”**：它负责解析 `.cu`，把不同部分分发给不同编译器处理，然后把结果合并。
> 

### nvcc 在 `.cu` 文件里要处理两类代码

一个 `.cu` 文件通常混在一起写：

- **Host 端 C++ 代码**（CPU 跑的）
- **Device 端 CUDA Kernel**（GPU 跑的 `__global__` / `__device__`）

nvcc 会做类似的拆分：

- host 代码 → 交给 **g++/clang++**
- device 代码 → 交给 **CUDA 的 device 编译器链**，生成：
    - **PTX**（中间表示，像 GPU 汇编）
    - 或 **SASS**（最终 GPU 指令，面向具体架构 sm_86 等）

所以你经常会看到 nvcc 命令里带着：

```bash
-gencode=arch=compute_86,code=sm_86
```

这意思是：给某个 GPU 架构生成对应的代码。

## 4. ninja：不编译代码，只负责“跑命令、并行、增量构建”

ninja 这玩意非常“朴素”：

- 它不懂 C++ 语法，也不懂 CUDA
- 它只读一个文件：`build.ninja`
- 里面写着一堆规则：**这个目标由哪些输入生成，用什么命令生成**

比如：

- `sampling.o` 由 `sampling.cpp` 生成，命令是 `g++ ...`
- `sampling_gpu.o` 由 `sampling_gpu.cu` 生成，命令是 `nvcc ...`

ninja 的强项是：

- **非常快**
- **并行构建**容易
- **增量构建**准确（改哪个文件就重编哪个）

所以当你看到：

```bash
ninja: build stopped: subcommand failed.
```

真正失败的命令通常在上面：可能是 `nvcc` 或 `g++` 的报错。ninja 只是“看到有人失败了，于是停工”。

## 5. 真实场景：PyTorch CUDA Extension是怎么把三者串起来的？

以我遇到的 `CUDAExtension(...)` 为例，它的典型流程是：

1. Python setup / build 系统（setuptools + PyTorch cpp_extension）
2. 生成 `build.ninja`
3. ninja 按规则并行执行：
    - `.cpp` → 用 g++ 编译成 `.o`
    - `.cu` → 用 nvcc 编译成 `.o`（nvcc 内部也会调用 g++）
4. 最后用 g++（或 nvcc 驱动）把 `.o` 链接成 Python 可 import 的 `.so`

所以在同一次编译里同时看到 g++ 与 nvcc 是正常的，而且 ninja 负责把它们“跑起来”。

## 6. 三者常见报错各自意味着啥？

### A) g++ 报 `xxx.h: No such file`

- 头文件 include 路径问题（`I` / `CPATH`）

### B) g++ 链接时报 `cannot find -lxxx`

- 链接路径问题（`L` / `LIBRARY_PATH`）

### C) nvcc 报 `cuda_runtime.h: No such file`

- CUDA 头文件路径没进 nvcc 的 include 搜索路径
- 或你以为装了 toolkit，其实只装了 nvcc/部分组件

### D) ninja 报 `build stopped`

- ninja 自己不负责报错根因，它只是告诉你：有人失败了
    
    真正错误在它上面那段 `nvcc ...` 或 `c++ ...` 的输出里。
    

## 7. 一句话总结

- **g++/clang++**：把 C/C++ 编译成目标文件并完成链接
- **nvcc**：CUDA 编译的“总调度”，`.cu` 里 host 交给 g++，device 交给 CUDA 编译链，再汇总链接
- **ninja**：构建执行器，只负责按规则并行跑命令，不负责理解代码