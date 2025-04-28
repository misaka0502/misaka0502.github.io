---
title: Ubuntu安装不同版本的GCC/G++并切换
date: 2025-04-29 00:40:21
tags: 
    - Linux
banner_img: /images/bg/laoma2.jpg
---
<script src="https://fastly.jsdelivr.net/gh/misaka0502/live2d-widget@V0.2/autoload.js"></script>
<!-- <script src="/live2d-widget/autoload.js"></script> -->

有时会遇到需要在同一台机器上使用不同版本的GCC/G++编译器的情况，比如在编译不同版本的项目时，或者在使用某些特定的库时。
假设系统中已经安装了gcc13和g++13，以安装gcc11、g++11为例：

```bash
sudo apt install gcc-11 g++-11
```

顺利的话，系统中就会存在两个版本的gcc和g++。在终端中输入：

```bash
ls /usr/bin/gcc*
ls /usr/bin/g++*
```

应该会列出系统中所有版本的gcc和g++

但是此时默认还是使用13，想要切换版本，需要使用`update-alternatives` 命令。

`update-alternatives` 是Debian及其衍生发行版（如Ubuntu）中一个非常重要的命令行工具，用于**管理系统中多个提供相同通用功能程序版本之间的符号链接**。（From Gemini 2.5 Pro）

简单来说，当你的系统上安装了同一个软件的多个版本，`update-alternatives` 可以帮助你方便地设置当你在命令行输入一个通用命令（如java）时，系统默认应该执行哪一个具体的版本。

1. 首先查看`update-alternatives`中有没有安装我们需要的gcc和g++版本：
   
    ```bash
    sudo update-alternatives --display gcc
    sudo update-alternatives --display g++
    ```
    
    输出会告诉你当前默认指向哪个路径（例如 /usr/bin/gcc-11），以及所有可用的版本路径及其对应的优先级。
    
2. 如果gcc和g++没有自动被注册，则需要手动添加：
   
    ```bash
    sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-13 120
    sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/g++-13 120
    
    sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-11 110
    sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/g++-11 110
    ```
    
    以第一条命令为例解释一下参数：
    
    - /usr/bin/gcc: 用户使用的通用命令链接。
    - gcc: update-alternatives 内部的组名。
    - /usr/bin/gcc-13: 你手动安装的 gcc 的实际路径。
    - 120: 你为这个版本设定的优先级（需要根据系统已有优先级来定，越高越优先）。这里我将系统原本自带的版本的优先级设置得最高，方便切换回来。
    
    然后再次用`--display` 确认是否成功添加。此外，还可以用`--slave` 参数将gcc和g++关联，操作量少一些。
    
3. 切换默认版本（交互式配置），在终端中输入：
   
    ```bash
    sudo update-alternatives --config gcc
    ```
    
    系统会列出所有已注册的 gcc 候选项，并带有编号：
    
    ```bash
    There are 2 alternatives for gcc (providing /usr/bin/gcc).
    
      Selection    Path              Priority   Status
    ------------------------------------------------------------
    * 0            /usr/bin/gcc-13    120       auto mode
      1            /usr/bin/gcc-11    110       manual mode
    
    Press <enter> to keep the current choice[*], or type selection number:
    ```
    
    输入你想要设置为默认版本的编号（例如输入 1 选择 gcc-11），然后按回车。
    
    **重要：** 你通常需要**同时**为 g++ 设置对应的版本，以保持 C 和 C++ 编译器版本一致：
    
    ```bash
    sudo update-alternatives --config g++
    ```
    
    同样，在列出的选项中选择与你刚才为 gcc 选择的**相同版本号**的 g++ 候选项（例如 g++-11）。
    
4. 验证切换结果：
   
    ```bash
    gcc --version
    g++ --version
    ```
    
    输出应该显示你刚刚选择的那个版本号。
    
5. 最好在临时切换gcc和g++编译你的工作完成后及时将版本再切换回系统原本的默认版本：
   
    ```bash
    sudo update-alternatives --auto gcc
    sudo update-alternatives --auto g++
    ```
    
    `--auto` 自动模式下会自动选择优先级最高的版本。
    

通过以上步骤，就可以有效地使用 update-alternatives 来管理和切换系统默认的 GCC/G++ 编译器版本了。