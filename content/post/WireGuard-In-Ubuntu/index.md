---
title: "搭建 WireGuard 网络"
description: 搭建 WireGuard 服务器端, 附带有客户端的一些配置
date: 2022-11-29T19:00:24+09:00
hidden: false
comments: true
draft: false
toc: false
tags: 
  - Linux
---


本文介绍在一台 `Ubuntu 22.04` 的服务器下搭建 WireGuard 服务器端, 附带有客户端的一些配置.

## 安装

WireGuard 已经集成在 5.6 及以上版本的 Linux Kernel 中, `Ubuntu 22.04` 系统下直接安装即可

```
sudo apt update
sudo apt install wireguard
```

## 生成密钥对

```
cd /etc/wireguard
wg genkey | tee s_private.key | wg pubkey > s_public.key
wg genkey | tee c_private.key | wg pubkey > c_public.key
```

此时会在目录下生成四个文件, 具体说明如下

| 文件名 | 文件说明 |
| --- | --- |
| s_private.key | 服务器端私钥 |
| s_public.key | 服务器端公钥 |
| c_private.key | 客户端私钥 |
| c_public.key | 客户端公钥 |

## 配置 WireGuard 服务器端

具体需要各端准备的变量有以下这些

- 服务器端: 公网 IP, 内网 IP, 公网端口, 公钥, 私钥, 网络接口
- 客户端: 内网 IP, 私钥, 公钥

整理为如下表格

|  | 服务器端 | 客户端 |
| --- | --- | --- |
| 公网IP | 233.233.233.233 |  |
| 内网IP | 192.168.1.1 | 192.168.1.2 |
| 端口配置 | udp/6666 |  |
| 私钥文件 | s_private.key | c_private.key |
| 公钥文件 | s_public.key | c_public.key |
| 网络接口 | eth0 |  |


### 编写配置文件

文件位置为 `/etc/wireguard/wg0.conf`

```
[Interface]
Address = 192.168.1.1
DNS = 223.6.6.6
ListenPort = 6666
PrivateKey = s_private.key
PostUp   = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -A FORWARD -o wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -D FORWARD -o wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
PublicKey = c_public.key
AllowedIPs = 192.168.1.2
```

其中 `Interface` 可以大致理解为本机的配置, 而 `Peer` 是面向外部的配置. 有一些参数需要按照实际情况调整, 密钥需要替换为实际的字符串

### 其他服务器配置

在服务器端启动 IP 转发

```
sysctl net.ipv4.ip_forward=1
```

另外, 可能需要云主机上配置开放对应的 UDP 端口

### 启动与停止

配置完成后, 可以使用如下命令, 分别是启动, 查看状态, 停止

```
wg-quick up wg0
wg show wg0
wg-quick down wg0
```

## WireGuard 客户端配置

### 编写配置文件

文件命名为 `client.conf`

```
[Interface]
PrivateKey = c_private.key
Address = 192.168.1.2
DNS = 223.6.6.6

[Peer]
PublicKey = s_public.key
Endpoint = 233.233.233.233:6666
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

也是要根据实际情况做相应的修改, 密钥替换为实际的字符串

### 其他客户端配置

可以使用如下命令在命令行生成二维码, 方便手机扫描

```
qrencode -t ansiutf8 < client.conf
```
