---
title: Ubuntu根目录空间占用过高，通过清理系统日志文件进行清理
date: 2025-09-05 19:31:52
tags:
    - Linux
    - Ubuntu
---
<script src="https://fastly.jsdelivr.net/gh/misaka0502/live2d-widget@V0.2/autoload.js"></script>
<!-- <script src="/live2d-widget/autoload.js"></script> -->

# Ubuntu根目录空间占用过高，通过清理系统日志文件进行清理

## 问题发现

今天突然发现 `/` 目录占用达到了100%，需要清理一下磁盘，进入`/` 目录一看，发现`/var/log` 占用了55G磁盘，再进`log` 中查看，发现`syslog` 就占用了50G。

## 处理方法

**1.临时清空日志**

```bash
sudo truncate -s 0 /var/log/syslog
```

由于`syslog`是系统日志文件，不能直接删除。上述命令可以直接清空文件内容，马上释放空间，而不是删除文件本身。这样文件大小变为 0，但文件仍存在，系统还能继续写入。

**2.使用 logrotate 管理日志**

Ubuntu 自带 **logrotate** 来自动清理旧日志。配置文件在 `/etc/logrotate.conf` 和 `/etc/logrotate.d/` 下。

- 手动执行一次轮转：
  
    ```bash
    sudo logrotate -f /etc/logrotate.conf
    ```
    
- 可以设置日志保留时间、压缩策略等（比如只保留 7 天，超出的删除）。

这样`/` 目录一下清出来几十G空间。

## 新的问题

在清理完`/` 目录后过了一小会，系统又警告我`/` 目录满了，一检查发现确实又满了，说明可能有某个服务在一直写入log。

查看`syslog` :

```bash
sudo tail -f /var/log/syslog
```

发现日志中在不断刷新一大堆update-notifier-crash相关的日志:

```bash
Sep 5 15:23:24 rlg3 update-notifier-crash[139502]: [139502:0100/000000.782142:ERROR:content/zygote/zygote_linux.cc:245] Error reading message from browser: Socket operation on non-socket (88) 
Sep 5 15:23:24 rlg3 update-notifier-crash[139502]: [139502:0100/000000.792141:ERROR:content/zygote/zygote_linux.cc:245] Error reading message from browser: Socket operation on non-socket (88) 
Sep 5 15:23:24 rlg3 update-notifier-crash[139502]: [139502:0100/000000.792154:ERROR:content/zygote/zygote_linux.cc:245] Error reading message from browser: Socket operation on non-socket (88)
```

- **`update-notifier-crash`** 是 Ubuntu 自带的一个程序，用来弹出「系统崩溃报告」或「有更新可用」的通知。
- 它调用了 **webkit / chromium 内核的组件**，而这些组件有 bug（zygote 进程通信失败），于是疯狂刷日志。
- 这就是导致 `/var/log/syslog` 快速膨胀的“罪魁祸首”。

## 解决方法

**1.直接禁用 `update-notifier-crash`**

如果你不需要自动弹更新通知，可以把它关掉：

```bash
sudo apt remove update-notifier-common
```

或者干脆禁用 service：

```bash
sudo systemctl stop update-notifier-crash.service
sudo systemctl disable update-notifier-crash.service
```

**2.屏蔽它的日志写入**

如果你还想保留它，但不想刷 `syslog`，可以用 `rsyslog` 规则屏蔽：

编辑 `/etc/rsyslog.d/ignore-update-notifier.conf`，加入以下内容：

```
:programname, isequal, "update-notifier-crash" stop
```

然后重启rsyslog:

```bash
sudo systemctl restart rsyslog
```

写入这个配置之后可能会无法生效，这可能是因为`rsyslog`的配置加载顺序导致规则被覆盖。

- **rsyslog 会按文件名的字典序依次加载 `/etc/rsyslog.d/` 下的 `.conf` 文件**，然后再加载 `/etc/rsyslog.conf` 中的配置。
- 当多个规则作用于同一个日志消息时，先匹配的规则会生效，如果后面有规则覆盖或把消息重新写入 `syslog`，就可能“打回原样”。

**总结就是先读取的配置会生效，读取顺序取决于文件名。**

所以，可以尝试更改上面的配置文件名:

```bash
sudo mv /etc/rsyslog.d/ignore-update-notifier.conf /etc/rsyslog.d/01-ignore-update-notifier.conf
```

再重启`rsyslog`，这样就能生效了。