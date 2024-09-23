---
title: 记录在Linux服务器上配置Clash+yacd dashboard
date: 2024-09-23 16:43:38
tags: Linux
index_img: /img/hina2.jpg
banner_img: /img/ako1.png
banner_img_height: 100
---
日志参考: [Linux 服务器安装 Clash代理](https://blog.myxuechao.com/post/36)，感谢作者。

# 一、安装与配置Clash

由于Clash的Github仓库已经被ban了，获取方法各凭本事，这里就不赘述了。

1. 创建文件夹：

```bash
mkdir clash
cd clash
```

2. 下载、解压、安装
   1. 以clash-linux-amd64-latest.gz为例，下载放进clash文件夹里并解压：`gunzip  clash-linux-amd64-latest.gz`
   2. 为了方便将解压后的文件重命名为 `clash`：`mv clash-linux-amd64-latest clash`
   3. 给予执行权限：`chmod +x clash`
   4. 启动Clash：`./clash -d .`, Clash会自动生成 `config.yaml`配置文件，将配置文件内容替换成自己订阅后得到的配置文件即可。
      - `./clash -d .` 的含义是：启动 clash 程序，并将当前目录作为其工作目录或配置目录。这通常用于指向包含配置文件或其他必要资源的目录。

> **完成以上步骤之后理论上终端会输出代理相关内容，但是此时还没有代理功能，需要配置系统代理，让流量走Clash。**

# 二、配置系统代理

**1. 临时代理配置：**

```bash
export http_proxy=http://127.0.0.1:7890
export https_proxy=http://127.0.0.1:7890
```

**2. 永久代理配置：**

1. 编辑 `~/.bashrc`文件：`vim ~/.bashrc`
2. 在文件末尾添加：

```bash
  export http_proxy=http://127.0.0.1:7890
  export https_proxy=http://127.0.0.1:7890
```

3. 保存并退出，使配置生效：`source ~/.bashrc`

**3. 一键开关代理：**

1. 编辑 `~/.bashrc`文件：`vim ~/.bashrc`
2. 在文件末尾添加：

```bash
   # 开启代理
   function proxy_on(){
       export all_proxy=socks5://127.0.0.1:7890
       export http_proxy=http://127.0.0.1:7890
       export https_proxy=http://127.0.0.1:7890
       echo -e "已开启代理"
   }

   # 关闭代理
   function proxy_off(){
       unset all_proxy
       unset http_proxy
       unset https_proxy
       echo -e "已关闭代理"
   }
```

3. 保存并退出，使配置生效：`source ~/.bashrc`
4. 需要代理时，在终端输入 `proxy_on`即可开启代理，不需要代理时，在终端输入 `proxy_off`即可关闭代理。
5. **测试代理是否生效：**
   终端输入:

```bash
curl www.google.com
wget www.google.com
```

# 三、安装与配置yacd dashboard

1. 切到clash的目录下（与config.yaml同一层），下载：

```bash
wget https://github.com/haishanh/yacd/archive/gh-pages.zip
```

2. 解压并重命名：

```bash
unzip gh-pages.zip
mv yacd-gh-pages/ dashboard/
```

3. 修改clash/config.yam，主要注意一下几个配置:

```yaml
external-controller: 0.0.0.0:9090
external-ui: dashboard
secret: 123456
```

- external-controller：外部控制端口，用于面板控制(前端页面的端口)
- external-ui：本地控制页面的源码（前端面板的路由）
- secret：用于yacd dashboard的登录密码，可以自行设置

4. 访问yacd dashboard：
   1. 浏览器访问 `http://yacd.haishan.me/`
   2. 按照如下方式填写：
      ![yacd](img\yacd.png)
   3. 点击add后即可进入面板：
      ![dashboard](img\dashboard.png)
      即可像使用cfw或者clash-verge等客户端一样监控和管理代理了

记录可能遇到的问题：

- 如果chrome无法登录yacd dashboard，可以尝试将网站设置的 `不安全内容`设为 `允许`
  ![网站设置](img\setting1.png)

同时还可以使用守护进程Clash自启动以及后台运行，不过我没用到所以暂时不记录了
