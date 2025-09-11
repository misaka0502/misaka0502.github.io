---
title: Ubuntu远程服务器无法正常git pull
date: 2025-09-11 10:00:50
tags:
    - Ubuntu
    - git
---
<script src="https://fastly.jsdelivr.net/gh/misaka0502/live2d-widget@V0.2/autoload.js"></script>
<!-- <script src="/live2d-widget/autoload.js"></script> -->

# Ubuntu远程服务器无法正常git pull

## 问题

今天在服务器上运行git pull发现无法拉取，输出以下信息:

```bash
Missing or invalid credentials.
Error: connect ECONNREFUSED /run/user/1028/vscode-git-363d95683a.sock
    at PipeConnectWrap.afterConnect [as oncomplete] (node:net:1611:16) {
  errno: -111,
  code: 'ECONNREFUSED',
  syscall: 'connect',
  address: '/run/user/1028/vscode-git-363d95683a.sock'
}
Missing or invalid credentials.
Error: connect ECONNREFUSED /run/user/1028/vscode-git-363d95683a.sock
    at PipeConnectWrap.afterConnect [as oncomplete] (node:net:1611:16) {
  errno: -111,
  code: 'ECONNREFUSED',
  syscall: 'connect',
  address: '/run/user/1028/vscode-git-363d95683a.sock'
}
```

排查了一遍，代理是正常开启的，SSH key是正常配置的，`wget` 也能访问github，唯独`git pull` 无法正常工作。尝试执行`ssh -T [git@github.com](mailto:git@github.com)` 没有反应，应该是连接超时；执行`ssh -vT [git@github.com](mailto:git@github.com)` 有以下输出:

```bash
OpenSSH_8.2p1 Ubuntu-4ubuntu0.13, OpenSSL 1.1.1f  31 Mar 2020
debug1: Connecting to github.com [140.82.112.5] port 22.
```

这说明服务器正尝试通过22端口访问github，而问题就出在这，因为很多服务器（比如校园机房、公司服务器）会默认封锁 22 端口，这就导致服务器 **22端口出不了网。**

总结一下现象：

- 代理正常，能通过`wget` 访问github
- 执行 `ssh -T git@github.com` 没有输出或超时。
- 使用 `ssh -vT git@github.com` 调试发现 **22 端口被防火墙/网络策略阻断**。
- 导致 `git pull` / `git push` 无法通过 SSH 认证。

## 解决方法：改用SSH 443端口

以下操作的前提是**已经将远程仓库设置为SSH链接，且已经配置好SSH Key**！

1. 编辑SSH配置文件:
   
    ```bash
    vim ~/.ssh/config
    ```
    
2. 添加以下内容:
   
    ```bash
    Host github.com
      Hostname ssh.github.com
      Port 443
      User git
      IdentityFile ~/.ssh/id_ed25519
    ```
    
    - `IdentityFile` 指定的是私钥路径。
3. 测试连接:
   
    ```bash
    ssh -T git@github.com
    ```
    
    如果成功，会提示:
    
    ```bash
    Hi <用户名>! You've successfully authenticated...
    ```
    
    第一次配置可能会有一个安全信任的设置:
    
    ```bash
    The authenticity of host '[ssh.github.com]:443 ([xxx]:443)' can't be established.
    ECDSA key fingerprint is SHA256:xxx.
    Are you sure you want to continue connecting (yes/no/[fingerprint])?
    ```
    
    输入`yes` 即可。
    
4. 再次拉取代码:
   
    ```bash
    git pull
    ```
    

这样应该就能成功拉取了。

## 总结

- **问题根源**：22 端口被封锁，SSH 默认无法连 GitHub。
- **解决思路**：要么换 HTTPS + Token，要么配置 SSH 走 443 端口。
- **推荐做法**：在远程服务器上配置 SSH 走 443，免去每次输入 Token 的麻烦。