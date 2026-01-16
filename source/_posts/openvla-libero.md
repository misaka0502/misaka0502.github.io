---
title: 在A100平台集群上部署OpenVLA与LIBERO仿真环境的踩坑及解决方法记录
date: 2026-01-16 12:30:24
tags:
    - Python
    - Linux
    - Ubuntu
    - Pytorch
---
<script src="https://fastly.jsdelivr.net/gh/misaka0502/live2d-widget@V0.2/autoload.js"></script>
<!-- <script src="/live2d-widget/autoload.js"></script> -->

## 1. 问题背景

在尝试运行OpenVLA的LIBERO评测脚本时，遇到了从环境导入到编译安装，再到底层图形驱动的一系列问题。

运行环境：

- 硬件：NVIDIA A100
- 任务：OpenVLA 7B 策略在LIBERO仿真环境中的评估
- 软件：PyTorch + MuJoCo (Robosuite)

## 2. 主要问题及解决方案

### 问题一：模块搜索路径失效 (ModuleNotFoundError)

**问题描述**：环境中明明安装了libero，`pip show` 也能看到包的信息，但是`import` 时还是报错找不到模块：`No module named 'libero’`。

- 原因分析：在复杂的科研集群中，使用 `pip install -e .` (Editable mode) 往往会因为 Conda 环境路径优先级或权限问题，导致生成的 `.egg-link` 映射失败。Python 解释器在 `sys.path` 中找不到源码位置。
- 解决方式：**显式指定 PYTHONPATH**。这是最稳健的“暴力”手段，它绕过了所有复杂的 `pip` 映射逻辑，直接在系统层面告诉 Python 解释器去哪里找代码：
    
    ```bash
    export PYTHONPATH=$PYTHONPATH:/path_to_LIBERO
    ```
    

### 问题二：GitHub 依赖克隆超时 (Connection Timeout)

**问题描述**：安装OpenVLA时，其依赖项（如dlimp）通过Git链接在线下载，导致`Conection timed out` 。

**解决方法**：

1. 先通过镜像站地址手动安装依赖:
    
    ```bash
    pip install git+https://githubfast.com/moojink/dlimp_openvla.git
    ```
    
2. 修改源码：修改OpenVLA的`pyproject.toml` ，将对应的`git+https` 链接行删掉，直接缩写为`”dlimp”` 。这样`pip`在第二次安装的时候就会识别到本地已经安装该包，从而跳过克隆步骤。

### 问题三：FlashAttention编译环境缺失(nvcc/CUDA_HOME)

**问题描述**：OpenVLA推理脚本尝试调用`flash-attn`，但因系统未配置完整的CUDA编译链（缺失nvcc和CUDA_HOME变量）导致安装/调用失败。

**解决方法（无sudo）**：降级使用SDPA(nvcc/CUDA_HOME)。修改 `openvla_utils.py` 中的 `AutoModelForVision2Seq.from_pretrained` 调用，将 `attn_implementation` 参数设为 `"sdpa"`。

- 原因：SPDA是PyTorch内置的加速方案，在A100上性能已经非常接近FlashAttention，且无需任何C++编译，在不触及系统级环境的情况下快速跑通实验。

### 问题四：Headless渲染驱动（EGL初始化失败）

**问题描述**：`ImportError: Cannot initialize a EGL device display`。

**原因分析**：A100等纯计算卡环境通常只安装了计算驱动，缺少图形配置文件。MuJoCo默认找不到对应的NVIDIA EGL 供应商 （Vendor），导致离线渲染无法开启。

**解决方案（需sudo）**：手动注入NVIDIA驱动的ICD配置文件：

1. 创建配置文件：
    
    ```bash
    sudo mkdir -p /usr/share/glvnd/egl_vendor.d/
    sudo bash -c 'cat > /usr/share/glvnd/egl_vendor.d/10_nvidia.json <<EOF
    {
        "file_format_version" : "1.0.0",
        "ICD" : { "library_path" : "libEGL_nvidia.so.0" }
    }
    EOF'
    ```
    
2. 激活渲染配置：
    
    ```bash
    export MUJOCO_GL=egl
    export __GLVND_EXPOSE_VENU_PDEV_ONLY=1
    ```
    

上述步骤完成之后，OpenVLA的LIBERO评估脚本成功正常运行。