---
title: "PATH vs. LD_LIBRARY_PATH: Linux 开发里最容易踩坑的两条“路径”环境变量（以及一堆亲戚）"
date: 2026-01-14 19:37:58
tags:
    - Linux
---
<script src="https://fastly.jsdelivr.net/gh/misaka0502/live2d-widget@V0.2/autoload.js"></script>
<!-- <script src="/live2d-widget/autoload.js"></script> -->

> Intro: 你在Linux上敲`python`、`nvcc`、`gcc`等命令时，它们是怎么被找到的？你编译出来的程序运行时为什么会突然报错 `libxxx.so: cannot open shared object file` ？这些玄学的背后，几乎都和PATH、LD_LIBRARY_PATH以及它们的亲戚有关。
> 

这篇博客就把它们按“到底在找什么”讲清楚：

- PATH: 找可执行文件
- LD_LIBRARY_PATH: 找动态链接库
- 以及：头文件路径、链接路径、Python包路径、CMake查找路径…分别是谁管的

## 1. PATH: 我输入一个命令，它去哪里找？

### PATH管的是什么？

PATH是shell用来查找可执行文件（binary）的路径列表。

当你输入：

```bash
nvcc -V
```

系统就会按顺序在PATH里面的每个目录找`nvcc`这个可执行文件，**找到第一个就用它**。

你可以把它理解为：

> PATH=“命令在哪”
> 

### 快速验证

```bash
echo $PATH | tr ':' '\n'
which nvcc
type -a python
```

### 什么时候应该修改PATH？

- 你安装了一个新工具，但是系统找不到（`command not found`）
- 你想优先使用某个版本（比如conda env的`nvcc`/`python`/`cmake`）
- 你机器上同时安装了多个CUDA/多个Python，需要切换默认

典型做法：

```bash
export PATH=$CONDA_PREFIX/bin:$PATH
# 或
export PATH=/usr/local/cuda-11.8/bin:$PATH
```

## 2. LD_LIBRARY_PATH: 程序运行时，`.so`去哪找？

### LD_LIBRARY_PATH管的是什么？

`LD_LIBRARY_PATH` 是**动态链接器（dynamic loader）**，用来查找**共享库（shared libraries,** `.so`**）**的路径列表。

你运行一个程序或者import一个带C++/CUDA扩展的Python包时，它可能需要加载:

- `libcudart.so`
- `libtorch.so`
- `libstdc++.so`
- 以及各种 `libxxx.so`

如果找不到，就会出现经典报错;

```bash
error while loading shared libraries: libxxx.so: cannot open shared object file
```

你可以把它理解成：

> LD_LIBRARY_PATH = “运行时库在哪”
> 

### 快速验证

```bash
echo $LD_LIBRARY_PATH | tr ':' '\n'
ldd your_binary_or_extension.so | grep "not found"
```

### 什么时候应该修改LD_LIBRARY_PATH？

- 程序启动/导入时找不到`.so`
- 你在conda env里装了cuda runtime、cudnn、torch，运行时却在用系统库或缺库
- 你没有root权限，不能把库装进系统默认路径（/usr/lib, /usr/local/lib）

典型做法：

```bash
export LD_LIBRARY_PATH=$CONDA_PREFIX/lib:$LD_LIBRARY_PATH
# 或 CUDA 常见：
export LD_LIBRARY_PATH=$CUDA_HOME/lib64:$LD_LIBRARY_PATH
```

## 3. conda相关：`$CONDA_PREFIX` 是什么？为什么总出现 `$CONDA_PREFIX/bin` 和 `$CONDA_PREFIX/lib`？

这是很多人配环境时最常用，但也最容易糊涂的一块。简单说:

### A) `$CONDA_PREFIX`：当前激活环境的“安装根目录”

当你 `conda activate xxx` 之后，conda会设置：

- `CONDA_PREFIX=/home/.../miniconda3/envs/xxx`

也就是说：这个env里装的所有东西，基本都在这个目录树下面

你可以随时检查：

```bash
echo $CONDA_PREFIX
```

### B) `$CONDA_PREFIX/bin` : 这个环境的“命令入口”

`bin/` 里放的是可执行文件，比如：

- `python`, `pip`, `cmake`, `ninja`, `nvcc`（如果你用 conda 装了）
- 各种工具的 CLI

所以把它放到PATH最前面，就能保证你敲命令时优先使用当前env的版本：

```bash
export PATH=$CONDA_PREFIX/bin:$PATH
```

### C) `$CONDA_PREFIX/lib`：这个环境的“共享库仓库”

`lib/` 里放的是动态库/静态库，比如：

- `libcudart.so` / `libcublas.so` / `libcudnn.so`（如果 conda 装了 cuda runtime/cudnn）
- `libstdc++.so`（conda 里常见的一套运行时）
- 各种依赖库 `libxxx.so`

所以运行时缺库时，很多人会把它塞到 `LD_LIBRARY_PATH` :

```bash
export LD_LIBRARY_PATH=$CONDA_PREFIX/lib:$LD_LIBRARY_PATH
```

> 关键点：PATH负责找`nvcc`这个程序本体；LD_LIBRARY_PATH 负责找 nvcc 或你的程序在运行/链接时要用的 `.so`。
这也是为什么“看起来 nvcc 已经装了”但编译仍可能缺 `cuda_runtime.h` 或运行时缺 `.so` —— 它们属于不同阶段的查找逻辑。
> 

### D) `$CONDA_PREFIX/include`：头文件的常见位置

如果你安装了开发包（比如某些 `*-dev` 对应的 conda 包），头文件经常在：

- `$CONDA_PREFIX/include`
- 对 CUDA 来说还可能在：`$CONDA_PREFIX/targets/x86_64-linux/include`

比如：

```bash
find $CONDA_PREFIX -name cuda_runtime.h
# .../targets/x86_64-linux/include/cuda_runtime.h
```

这就很典型：头文件不一定在$CONDA_PREFIX/include，而是被分发到targets/.../include 这种更“CUDA 风格”的目录里

## 4. PATH vs LD_LIBRARY_PATH: 一句话区分

| 变量 | 负责找什么 | 发生在什么时候 | 常见症状 |
| --- | --- | --- | --- |
| `PATH` | 可执行文件（命令） | 你在 shell 里敲命令时 | `command not found` / 用错版本 |
| `LD_LIBRARY_PATH` | 动态库 `.so` | 程序启动、运行中加载库时 | `libxxx.so: cannot open...` |

**注意：这俩都不负责找头文件**`.h`**。**

头文件是编译器通过include路径找的，另有人管（下面提到）。

## 5. 其他常见“路径环境变量家族”（按用途分类）

### A) 头文件（headers）去哪找？——`CPATH/C_INCLUDE_PATH/CPLUS_INCLUDE_PATH`

当你编译C/C++/CUDA代码，遇到：

```bash
fatal error: xxx.h: No such file or directory
```

这属于**编译阶段include路径问题**，一般用这些变量（或编译参数`-I`）解决。

- `CPATH`: 对C和C++都生效（gcc/g++都会读）
- `C_INCLUDE_PATH`: 只对C（gcc）
- `CPLUS_INCLUDE_PATH`: 只对C++（g++）

示例：

```bash
export CPATH=$CUDA_HOME/targets/x86_64-linux/include:$CPATH
```

> 很多构建系统会自己加`-I …` ，但遇到“conda装了头文件却编译器找不到”时，CPATH很好救火。
> 

### B) 链接阶段去哪找库？——`LIBRARY_PATH`

你编译链接时如果报：

```bash
/usr/bin/ld: cannot find -lxxx
```

这是链接阶段找不到库（`.so`或`.a`）的问题。

`LIBRARY_PATH`影响的是编译/链接器在build时的查找路径（相当于补`-L …`）。

示例：

```bash
export LIBRARY_PATH=$CONDA_PREFIX/lib:$LIBRARY_PATH
```

对比一下：

- `LIBRARY_PATH`：build 时（链接器）
- `LD_LIBRARY_PATH`：run 时（动态加载器）

### C) 系统默认库搜索路径：`ldconfig` 与 `/etc/ld.so.conf`

这条你没有root权限时通常动不了，但理解它很重要。

动态连接器默认会在：

- `/lib`, `/usr/lib`, `/usr/local/lib`（以及架构相关目录）
- `ldconfig` 缓存的路径

里找库。

所以root用户常见“优雅解”是把库路径写入 `/etc/ld.so.conf.d/*.conf` 然后 `sudo ldconfig`，避免依赖 `LD_LIBRARY_PATH`。

但在服务器/无权限环境里，`LD_LIBRARY_PATH` 就是最常用的用户动态方案。

### D) Python模块去哪找？——PYTHONPATH

Python import时的搜索路径，跟`PATH` 没啥关系。

当你 `Import xxx` 找不到模块时，可以看：

```bash
python -c "import sys; print('\n'.join(sys.path))"
echo $PYTHONPATH
```

示例：

```bash
export PYTHONPATH=/home/me/my_project:$PYTHONPATH
```

> 注意：conda env的 `site-packages` 通常会自动进 `sys.path` ，一般不必手动添加，除非你要临时把某个目录当成包路径。
> 

## 6. 一个“对号入座”的排错速查表

### 你敲命令：报“command not found”

✅ 看 `PATH`

```bash
which nvcc
echo $PATH
```

### 编译时报“xxx.h: No such file”

✅ 看 include 路径（`CPATH` 或编译参数 `-I`）

```bash
echo $CPATH
# 或检查 build 命令行里有没有 -I/path/to/include
```

### 运行/导入时报“libxxx.so: cannot open”

✅ 看运行时库路径（`LD_LIBRARY_PATH` 或系统 ld 缓存）

```bash
echo$LD_LIBRARY_PATH
ldd your_binary_or_extension.so | grep "not found"
```

### Python import 报“No module named …”

✅ 看 `PYTHONPATH` / `sys.path`

```bash
python -c "import sys; print('\n'.join(sys.path))"
```

## 7. 经验总结

开发中最常见的坏结局是：你不断往 `LD_LIBRARY_PATH` / `PATH` 里面塞路径，最后环境能跑但是说不清为什么能跑。

更稳的做法是：

- 优先使用conda/venv的隔离（`$CONDA_PREFIX/bin`、`$CONDA_PREFIX/lib`）
- 编译时尽量让构建系统明确写出`-I`/`-L` （或用CMake/pip/torch extension 的标准方式）
- 需要长期生效的环境变量，集中写到一个脚本里（比如 `env.sh`），每次进入项目`source [env.sh](http://env.sh/)`