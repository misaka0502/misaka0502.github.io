---
title: 在一台计算机上运行多个torchrun任务
date: 2026-01-14 19:31:24
tags:
    - Python
    - Ubuntu
    - Pytorch
---
<script src="https://fastly.jsdelivr.net/gh/misaka0502/live2d-widget@V0.2/autoload.js"></script>
<!-- <script src="/live2d-widget/autoload.js"></script> -->

## 1. 问题

当我在一个终端中运行 `CUDA_VISIBLE_DEVICES=6,7 torchrun --nproc_per_node=2 xxx.py` ，然后打开了第二个终端运行 `CUDA_VISIBLE_DEVICES=3,4 torchrun --nproc_per_node=2 xxx.py` 时，产生了报错:

```bash
RuntimeError: The server socket has failed to listen on any local network address. The server socket has failed to bind to [::]:29500 (errno: 98 - Address already in use). The server socket has failed to bind to ?UNKNOWN? (errno: 98 - Address already in use).
```

原因在于，`torchrun` 默认使用 `29500` 端口。第二个任务必须指定不同的端口，否则会报 `Address already in use` 错误。

## 2. 避免端口冲突

因此，要在同一台机器上同时运行两个或多个 `torchrun` 任务，必须解决**端口冲突** 。解决方法也很简单: 

- **手动指定端口**：使用 `—master_port=29501` 或 `--rdzv_endpoint=localhost:端口号`。
- **自动分配端口**：将端口设置为 `0`，系统会自动找空闲端口。

此外，注意设置好`CUDA_VISIBLE_DEVICES` 。