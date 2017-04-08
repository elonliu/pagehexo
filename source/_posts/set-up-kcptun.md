---
title: 使用 kcptun 加速 Shadowsocks
mathjax: false
tags:
  - Shadowsocks
  - kcptun
  - VPS
  - Bandwagon
date: 2017-04-08 11:35:16
categories: Lifehacks
---

一直以来自己使用的 VPN 都是搭建在平民价位的搬瓦工 (Bandwagon) VPS 上的 Shadowsocks 代理，连接速度经常慢得感人。最近得知可以使用 kcptun 通过 UDP 来加速，加速效果非常理想，可以在 YouTube 上观看 1080P 视频不卡。因此动手搭建起来。

<!-- more -->

## 前期工作

首先需要租用一个 VPS 并在其上面搭建 Shadowsocks. 包括 VPS 选购，以及 Shadowsocks 的搭建，在[老高的博客](https://blog.phpgao.com/)上有非常详细的介绍。我的所有搭建工作（包括本文）基本都是参照其帖子，强烈有需要的朋友参考。

## 使用 kcptun

使用 kcptun 的详细过程，请参见老高的这篇帖子：
> [使用 KCPTUN 加速你的 VPS](https://blog.phpgao.com/kcptun.html)

重复的事情不用多说，重点分享一下自己搭建过程中的问题与经验。

### 搞清端口号
一般来讲这里面最绕的就是端口号，因为一共有4个端口号需要填写。当然老高的帖子里写得比较清楚了，但机械地填写还是容易填错，尤其当自己使用了与老高示例中不同的端口时。这时了解其原理，根据逻辑理解就容易理清头绪。

流量按照你配置的端口，可以连接成一个回路，这样才对。所以检查一下端口连接是否满足：
- Shadowsocks 客户端 `server port` = kcptun 客户端 `localaddr`
- kcptun 客户端 `remoteaddr` 端口 = kcptun 服务器端 `listen`
- kcptun 服务器端 `target` 端口 = Shadowsocks 服务器端 `server_port`

### 连接调试
在连接调试阶段不要把信息重定向到 `/dev/null`，这样可以看到实时连接信息。即便端口号确保无误的情况下，也有可能出现 `getsockopt: connection refused` 错误，请检查以下两方面：
- 服务器端与客户端是否使用了相同版本的 kcptun. 不同版本有可能带来错误
- 不同的 VPS 对于本地地址定义不同。请尝试更改 kcptun 服务器端配置里的 `target` 一项的本地地址，由 `127.0.0.1` 改为 `localhost` 或者 VPS 实际的 IP 地址。我的搬瓦工 VPS 改成了实际 IP 地址才行。

### 使用 Supervisor
如果 Shadowsocks 使用了 Supervisor 管理，那把 kcptun 也加入其中是个不错的选择，可以一并管理，并通过网页界面远程管理、查看状态。

> 关于使用 Supervisor 管理 Shadowsocks 相关操作，请参考老高的另一篇帖子：
> [使用 Supervisor 托管 Shadowsocks](https://blog.phpgao.com/supervisor_shadowsocks.html)

在 Supervisor 配置文件（如 `/etc/supervisord.conf`）末尾添加：
```
[program:kcptun]
command = /path/to/server_linux_amd64 -c /path/to/server.json
user = root
autostart = true
autoresart = true
```

更新配置后重新运行 Supervisor:
```bash
supervisorctl reload
```

### Windows GUI
在 Windows 下运行一个命令行的客户端比较烦人。GitHub 上有人开发了[图形界面工具](https://github.com/GangZhuo/kcptun-gui-windows/releases)，也比较方便。

## 效果测试
加速后观看 YouTube 视频的缓冲速度有了非常明显的提升，720p 或者 1080p 基本上都没什么问题。所以非常建议使用！