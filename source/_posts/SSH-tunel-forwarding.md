---
title: 使用SSH 隧道转发绕过服务器网络封锁 / Windsurf连接远程服务器时加载不出Cascade
date: 2026-01-14 19:33:24
tags:
    - Linux
    - Ubuntu
    - Debug
    - Windusrf
---
<script src="https://fastly.jsdelivr.net/gh/misaka0502/live2d-widget@V0.2/autoload.js"></script>
<!-- <script src="/live2d-widget/autoload.js"></script> -->

## 1. 问题描述

- 在远程服务器上进行开发的时候，突然发现Windsurf加载不出Cascade的对话框了，不管怎么刷新窗口、杀死进程都没用，Cascade窗口只会一个Logo，没有任何反应。但是在本地计算机上，Windsurf完全正常。
- 此外，运行clash进程的终端中，发现日志中频繁刷出`i/o timeout` 或 `connect error` 。

## 2. 故障排查与定位

我在远程服务器上按照以下几个步骤进行检查：

1. 打开yacd面板，进行连通性测试，所有节点都没有反应
2. 测试外网连通性，也连接不上

再加上clash日志中频繁刷出的`i/o timeout`所，以初步认定是代理出问题了。

## 3. 问题修复

尝试了很多方法之后，最后发现利用SSH隧道转发可以一定程度上解决这个问题。具体操作如下：

```bash
# 在本地计算机的终端中，执行下面的命令
ssh -p 远程服务器的端口号 -R 远程服务器的代理端口(例如7890):127.0.0.1:本地计算机的代理端口号(例如7897) [用户名]@[服务器IP]
```

如果成功，会顺利登入服务器，并且测试外网也能正常访问了，这说明SSH隧道转发已经成功建立了。

但是，到这里Windsurf的Cascade还是无法加载出来，一段时间后clash也正常了，Cascade还是加载不出来，说明并不完全是代理的问题。

## 4. 继续排查

### 检查远程Windsurf进程

```bash
ps aux | grep windsurf
```

此时发现异常现象：同时存在多套Windsurf Server：

```bash
~/.windsurf-server/bin/03467c19...
~/.windsurf-server/bin/97d7a9c6...
```

- **多个 extensionHost 同时运行**
- **多个 openai.chatgpt / codex 进程**
- 不同版本的 server / extension 相互竞争端口与 IPC

➡️ 这是 **Windsurf Remote 的典型冲突状态**

### 问题根因

> 多次升级 / 重连 Remote 后，
旧版 Windsurf Server 未被清理，
导致多个 server + extensionHost 同时运行。
> 

结果是：

- Cascade UI 连接到 server A
- OpenAI / Codex 实际跑在 server B
- IPC / WebSocket 断裂
- **前端无报错，但永远收不到响应（白屏）**

## 5. 最终解决策略

### 核心原则

> 不能在 Remote 连接状态下“强杀 server”
> 
> 
> 否则会被 Windsurf 自动重启。
> 

### 正确操作顺序

Step 1：在本地 Windsurf 断开 Remote

```
Cmd + Shift + P
→ Remote-SSH: Kill Remote Server
```

Step 2：确认远程 server 已完全停止

```bash
ps aux | grep windsurf
```

应只剩 `grep` 本身。
Step 3：彻底清理远程残留（**注意，此操作会把对话数据、相关配置也一并删除，请确认是否能够备份！**）

```bash
rm -rf ~/.windsurf-server
rm -rf ~/.codeium/windsurf
```

### Step 4：重新连接 Remote

- Windsurf 自动：
    - 上传单一版本 server
    - 重装扩展
    - 重建 IPC / 端口绑定

## 6. 结果与副作用

### 成功结果

- Cascade 在 Remote 环境中 **正常加载**
- 白屏问题彻底消失

### 预期副作用

- **Remote 侧 Cascade 历史对话全部丢失**

原因：

- Cascade 对话与 MCP 状态存储在：
    
    ```
    ~/.windsurf-server/
    ```
    
- 清理目录 = 清空远程状态

没有尝试在删除之前备份对话数据，不知道能不能行。

## 7. 经验教训

### Windsurf Remote 的关键特性

- Remote server 不会自动清理旧版本
- Cascade / MCP 状态与远程server绑定

## 8. 总结

这是一次典型的Remote状态飘逸问题，根本原因不是功能Bug，而是生命周期管理失败。通过”断连→清空→重建“，成功恢复系统一致性。