---
title: 使用 frp 做内网穿透
date: 2019-12-16 18:38:34
tags: 
  - frp
  - 树莓派
categories:
  - linux
---

最近在捣鼓树莓派，首先想达到的目标是：无论树莓派放在哪儿，只要能联网我就能 ssh 连上它。这首先就需要配置内网穿透了。

市面上有很多内网穿透方法，我选择了性能更好，配置更简单的 [Frp](https://link.zhihu.com/?target=https%3A//github.com/fatedier/frp) 。Frp 结构很简单，分为 frps/frpc 两个可执行程序，在有公网地址的 VPS 上运行 frps 然后在家中内网运行 frpc 就行。我有自己的阿里云 ecs，可以作为"桥梁"，frps 服务端配置就放在上面。

<!--more-->

## frp

[GitHub](https://github.com/fatedier/frp/blob/master/README_zh.md)

[frps 完整配置文件](https://github.com/fatedier/frp/blob/master/conf/frps_full.ini)

[frpc 完整配置文件](https://github.com/fatedier/frp/blob/master/conf/frpc_full.ini)

## 配置步骤

### 服务端配置

```bash
# 1. 安装 frp（linux）（下载地址从 https://github.com/fatedier/frp/releases 上找，可通过 uname -a 命令查看机器内核版本来决定使用哪个下载链接）
wget https://github.com/fatedier/frp/releases/download/v0.30.0/frp_0.30.0_linux_amd64.tar.gz

# 创建目录
mkdir -p /usr/local/frp
mv frp_0.30.0_linux_amd64.tar.gz /usr/local/frp

# 解压
tar -zxvf frp_0.30.0_linux_amd64.tar.gz
cd frp_0.30.0_linux_amd64

# frpc、frpc.ini、frpc_full.ini 为客户端配置文件，frps、frps.ini为服务端配置文件，此处删除客户端配置文件
rm frpc*
```

```bash
# 配置 frps.ini
vi frps.ini 

[common]
   bind_port = 7000            # 与客户端绑定的进行通信的端口，两端需一致
   token = 1234567             # token 需要服务端和客户端保持一致
```

```bash
# 服务端启动
./frps -c ./frps.ini
```

### 客户端配置

安装步骤与服务端一样，只是解压后要删除的是 frps、frps.ini、frps_full.ini

```bash
# 配置 frps.ini
vi frpc.ini

   [common]
   server_addr = ***.***.***.***   # 服务端（公网服务器）IP
   server_port = 7000              # 与服务端bind_port一致
   token = 1234567                 # token 需要服务端和客户端保持一致
   
   [ssh]
   type = tcp                      # 连接协议
   local_ip = 127.0.0.1            # 客户端（内网服务器）IP
   local_port = 22                 # ssh默认端口号
   remote_port = 12580             # 自定义的访问内部ssh端口号，任意一个大于 1024 且没使用的端口号都可以
```

```bash
# 客户端启动
./frpc -c ./frpc.ini
```

## 验证连接

```bash
ssh user@[server_addr] -p [remote_port]
```

