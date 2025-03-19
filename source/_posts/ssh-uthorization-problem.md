---
title: 通过SSH克隆Github仓库时报错The authenticity of host 'github.com' can't be established.
date: 2025-03-19 13:09:31
tags:
    - Linux
banner_img: /images/bg/hina5.png
---
<script src="https://fastly.jsdelivr.net/gh/misaka0502/live2d-widget@V0.2/autoload.js"></script>
<!-- <script src="/live2d-widget/autoload.js"></script> -->

今天在服务器上git clone时出现错误：

```bash
git@github.com: Permission denied (publickey).
fatal: Could not read from remote repository.
```

一看不知道为什么之前配置的SSH Key被删掉了，正好借此机会记录一下这个问题的解决。

出现这个问题说明Git无法同通过SSH访问Github，很大的可能是因为SSH Key配置有问题。

### 1.检查SSH Key是否已生成

检查目录~/.ssh/下是否有SSH Key：

```bash
ls ~/.ssh/id_rsa.pub
```

如果文件不存在，需要生成新的SSH key；如果文件存在，可以跳过第二步

### 2.生成新的SSH Key

如果没有SSH Key，可以用以下命令生成：

```bash
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
```

**注意：**

- 替换your_email@example.com为Github账户绑定的邮箱
- 运行后会提示存储路径，默认即可（~/.ssh/id_rsa）
- 可能会要求输入密码，可以直接跳过

参数：

- -t(Type)：指定要生成的密钥类型，-t rsa表示生产RSA密钥
- -b(Bits)：指定密钥长度（比特数），-b 4096表示生成4096-bit长度的RSA密钥

### 3.将SSH Key添加到SSH代理（可选）

运行：

```bash
eval "$(ssh-agent -s)"  # 启动 ssh-agent
ssh-add ~/.ssh/id_rsa    # 添加 SSH key
```

### 4.将SSH Key添加到Github

1. 运行以下命令，复制公钥：
2. 打开Github，进入**Settings>SSH and GPG keys**
3. 打开**New SSH Key**，然后：
    - Title：随意添加
    - Keyzhantie刚刚复制的SSH公钥内容
    - 点击Add SSH Key

### 5.测试SSH连接

运行：

```bash
ssh -T git@github.com
```

如果正确配置，应该看到：

```bash
Hi your-github-username! You've successfully authenticated, but GitHub does not provide shell access.
```

之后应该就能正常通过SSH克隆仓库了