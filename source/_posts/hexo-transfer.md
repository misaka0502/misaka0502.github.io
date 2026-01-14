---
title: 从一台电脑迁移 Hexo 项目到另一台电脑：踩坑记录与完整解决方案
date: 2026-01-14 19:24:54
tags:
    - Linux
    - Ubuntu
---
<script src="https://fastly.jsdelivr.net/gh/misaka0502/live2d-widget@V0.2/autoload.js"></script>
<!-- <script src="/live2d-widget/autoload.js"></script> -->

在日常开发中，把Hexo博客工程从一台电脑迁移到另一台电脑上是非常常见的事情，例如：

- 更换电脑
- 在服务器/Ubuntu环境继续写博客
- 在多台设备之间同步维护博客

但是Hexo项目并不是clone下来就能直接跑的。

本文记录一次真实的迁移过程，以及其中遇到的问题、背后的原因和标准的解决方案，供后续参考。

## 一、背景说明

### 原始环境

- 一台已经正常运行Hexo博客的电脑
- 博客工程已经push到GitHub（包含package.json/package-lock.json）

### 新环境

- Ubuntu系统
- 从Github clone博客仓库
- 目标: 继续写博客并本地生成站点
- 已经按照Hexo的安装说明安装过了Hexo

## 二、问题一：`hexo g`报错 Cannot find module ‘hexo’

### 复现错误

在新机器上clone项目之后，直接执行：

```bash
hexo g
```

报错如下：

```bash
ERROR Cannot find module 'hexo' from '/home/xxx/projects/xxx.github.io'
ERROR Local hexo loading failed
```

### 问题原因分析

Hexo的加载逻辑是：

> 优先使用当前项目中本地的hexo，而不是全局hexo
> 

也就是说，Hexo项目在运行时依赖：

```bash
项目目录/
 ├── node_modules/
 │   └── hexo
 └── package.json
```

而Github仓库中：

- 不会提交`node_modules/`
- clone下来的只是源码

所以新机器上：

- 没有安装项目依赖
- 本地hexo不存在
- hexo CLI无法加载项目

### 正确解决方式

在项目根目录执行：

```bash
npm install
```

如果遇到依赖冲突，可使用：

```bash
npm install --force
```

完成后，项目结构会变成：

```bash
xxx.github.io/
 ├── node_modules/
 ├── package.json
 ├── package-lock.json
 ├── source/
 └── _config.yml
```

此时再运行`hexo g` ，第一阶段问题已经解决

## 三、问题二：`hexo-render-pandoc`全面报错

### 新的错误现象

依赖安装完成后，再次执行：

```bash
hexo g
```

出现大量错误：

```bash
[ERROR][hexo-renderer-pandoc] pandoc exited with code null.
```

并且：

- 所有`.md` 文件全部失败
- 包括 `_posts/` 和 `about/index.md`
- 错误来源：`hexo-renderer-pandoc`

### 根本原因：系统中没有安装pandoc

这是整个迁移过程中最容易被忽略的一点

关键事实：

- `hexo-render-pandoc`不是纯JavaScript渲染器
- 它本质上是一个Node.js封装器
- 实际渲染Markdown时，会调用：

```bash
pandoc
```

这个系统级可执行程序

如果系统中：

- 没有安装pandoc
- 或pandoc不在PATH中

那么Node调用就会直接失败，表现为：

```bash
pandoc exited with code null
```

### 解决方式

在新机器上安装pandoc即可：

```bash
sudo apt update
sudo apt install pandoc
```

验证安装是否成功：

```bash
pandoc --version
```

只要能正常输出版本号即可。

### 重新生成站点

```bash
hexo clean
hexo g
```

此时：

- 所有Markdown文件可正产渲染
- 博客构建成功
- 问题解决

## 四、总结跨机器迁移Hexo项目的标准流程（Checklist）

```bash
# 1. 基础环境
node -v        # 建议 >= 16
npm -v

# 2. 安装hexo
npm install -g hexo-cli

# 3. 系统级依赖（如果使用 pandoc）
sudo apt install pandoc
# 如果还有其他系统级依赖，可能也要针对性安装

# 4. 项目依赖
npm install # (或 npm install --force)

# 4. 构建
hexo clean
hexo g
hexo s
```

这次迁移中遇到的问题，本质可以总结为两类:

1. Node项目通用问题
    - `node_module`不会随Git迁移
    - 必须在新环境重新npm install
2. Hexo+pandoc的系统依赖问题
    - `hexo-render-pandoc`依赖系统级pandoc
    - npm安装成功≠系统环境就绪