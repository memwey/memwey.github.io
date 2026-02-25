+++
title = "在 OpenWrt 上配置 Tailscale 服务"
date = "2026-02-24T14:32:01+09:00"
updated = "2026-02-25T20:33:08+09:00"
draft = false
description = "在 OpenWrt 上配置 Tailscale 服务, 实现中日远端内网互联"
[taxonomies]
tags = ["Linux", "Computer Network"]
+++

[Tailscale](https://tailscale.com/) 是一个基于 WireGuard 协议的组网服务, 它可以实现跨平台的内网互联, 作为一个 Golang 编写的开源软件, 它提供了较为慷慨的免费服务如 DERP 等, 同时也为有更多需求的用户提供付费服务.

我的使用环境比较特殊, 家在佛山, 而我平时在东京租房上班, 使用 Tailscale 实现东京与佛山的设备互联, 方便通信和管理. 同时, 也可以实现一些需要特定网络环境的场景, 比如区域授权流媒体或者一些需要在特定网络环境下运行的应用.

在东京实际搭建的时候, 发现东京的主路由 K2P 性能实在孱弱, 一开始 ROM 不足以安装 Tailscale, 精简固件后勉强安装, 又 RAM 不足, 反复发生 OOM 的情况, 最终把一台一直吃灰的斐讯 N1 找了出来安装 Tailscale, 作为旁路路由处理 Tailscale 的流量. 最终形成了一个稍微复杂的网络结构, 佛山的主路由作为 Tailscale 网络的入口, 而东京的 N1 作为旁路路由, 处理 Tailscale 网络中的流量. 不过最终效果还是一样的, 即两地内网直接互访, 且均提供网络出口.

## 网络结构

| 地点 | 设备角色 | 设备系统 | Tailscale 节点用途 | 内网网段 | 内网IP | 设备型号 |
|------|----------|----------|----------------------|--------------| ---------| ---- |
| 佛山 | 主路由 | OpenWrt 24.10 | 内网入口, 提供局域网穿透 | 192.168.128.0/24 | 192.168.128.1 | 歌华链 GHL-R-001 |
| 东京 | 主路由 | OpenWrt 24.10 | 无 | 192.168.9.0/24 | 192.168.9.1 | 斐讯 K2P |
| 东京 | 旁路路由 | Armbian 26.2.1 trixie | 提供局域网穿透 | 192.168.9.0/24 | 192.168.9.235 | 斐讯 N1 |

当前使用的是 1.94.2 版本的 Tailscale 客户端. 网络结构中内网网段不应该重叠, 不然会导致冲突.

最终实现的网络拓扑是这样的:
```
    终端设备 (PC/手机/NAS)
               │
      东京 LAN (192.168.9.0/24)
               │
    ┌──────────┴──────────┐
    │                     │
  OpenWrt (192.168.9.1) ──┼─── [静态路由] ─── Armbian + Tailscale (192.168.9.235)
    │                     │                   ( 处理去往 192.168.128.0/24 的流量 )
    │                     └───────────────────┘
    │
    └─── 公网 (Internet) ─── Tailscale 网络 (DERP/P2P)
                │
                │
         OpenWrt + Tailscale (192.168.128.1)
                │
         佛山 LAN (192.168.128.0/24)
```

在东京侧, 所有 LAN 设备的默认网关仍是 192.168.9.1, 通过静态路由, 去往佛山的流量会被中转给 192.168.9.235, 最终通过 Tailscale 网络到达佛山的设备.

## 在 OpenWrt 上安装 Tailscale 及 Web 管理界面

OpenWrt 使用 opkg 作为包管理工具, 官方自带的软件源中 Tailscale 长期没有更新了, 所以我们需要从第三方软件源中安装 Tailscale. 同时这个 Tailscale 经过裁剪和优化, 更适合路由器这种内存和闪存资源有限的设备. 并且我们增加 Tailscale 的 LuCI Web 管理界面, 方便管理和配置.

下面提供的是命令行的过程, 主要是添加证书需要到设备的命令行中执行, 另外两项操作添加软件源和安装其实也可以在 LuCI Web 管理界面中完成. 这里需要注意, 添加第三方证书有一定的风险, 可能会有供应链攻击的风险.

### 安装 Tailscale 服务
添加证书
```
wget -O /tmp/key-build.pub https://gunanovo.github.io/openwrt-tailscale/key-build.pub && opkg-key add /tmp/key-build.pub
```
添加软件源
```
echo "src/gz openwrt-tailscale https://gunanovo.github.io/openwrt-tailscale" >> /etc/opkg/customfeeds.conf
```
安装
```
opkg update && opkg install tailscale
```

### 安装 Tailscale LuCI Web 管理界面
添加证书
```
wget https://Tokisaki-Galaxy.github.io/luci-app-tailscale-community/all/key-build.pub -O /tmp/key-build.pub && opkg-key add /tmp/key-build.pub
```
添加软件源
```
echo "src/gz tailscale_community https://Tokisaki-Galaxy.github.io/luci-app-tailscale-community/all" >> /etc/opkg/customfeeds.conf
```
安装
```
opkg update && opkg install luci-app-tailscale-community
```

## 配置 OpenWrt 上的 Tailscale 客户端

此时应该可以在 LuCI Web 管理界面中看到 服务 -> Tailscale 选项, 点击进入配置页面. 在设置中点击登录, 会跳转到 Tailscale 网站进行登陆授权, 把这台设备添加到 Tailscale 网络中.

回到路由器的 LuCI Web 管理界面的 Tailscale 选项卡中, 需要打开 `接受路由`, `通告出口节点` 这两个选项. `接受路由` 用于接收 Tailscale 网络中的子网路由配置, 以实现不同内网的直接通信. 而 `通告出口节点` 则可以让 Tailscale 网络中别的设备选择通过这台设备代理进行网络访问, 即这台设备可以作为代理出口.

随后需要填写 `通告路由`, 填写的值应该是主路由对应的内网网段, 比如我这里, 佛山的 GHL-R-001 内网网段是 192.168.128.0/24, 东京的 K2P 内网网段是 192.168.9.0/24.

## 配置 Armbian 上的 Tailscale 客户端

在 Armbian 上安装 Tailscale 客户端的过程与 OpenWrt 类似, 只是需要在 Armbian 上需要使用命令行操作. 最终运行的命令如下
```
tailscale up --accept-routes=true --accept-dns=true --advertise-routes=192.168.9.0/24 --advertise-exit-node
```

### 配置 Armbian 的转发

确认 ipv4, ipv6 转发都已开启, 即值为 1
```
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.conf
echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p /etc/sysctl.conf
```
配置防火墙规则, 确保 Tailscale 网络中的流量可以正确转发
```
iptables -t mangle -A FORWARD -o tailscale0 -p tcp -m tcp \
--tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu
```
并且固化以防止重启丢失
```
sudo netfilter-persistent save
```
### 在主路由上配置静态路由

在东京主路由 K2P 上, 需要配置旁路网关的静态路由, 以确保流量可以正确路由到 Tailscale 网络中的设备. 具体来说, 需要在主路由上添加一条路由规则, **到 192.168.128.0/24 的路由，下一跳为 192.168.9.235**.


## 配置 Tailscale 服务管理

登陆 [Tailscale 管理页面](https://login.tailscale.com/admin/machines), 可以看到设备已经添加到 Tailscale 网络中了. 还需要在这个页面的设备管理上, 点击 `Edit route settings...`, 勾选对应的 `Subnet routes` 和 `Exit node` 选项. 不同设备都需要操作. 同时如果设备长期在线且物理安全可控, 也建议选择 `Disable key expiry`.

这里可能会有人纠结为什么要配置两次. 其实是因为 Tailscale 网络的 P2P 零信任的特性, 将节点状态的宣告和承认分开了. 宣告是节点将自己的路由信息告诉 Tailscale 网络, 而承认是 Tailscale 网络承认这个信息, 然后将这个节点宣告的路由信息通知给网络中的其他节点, 所以需要在节点和 Tailscale 管理页面上都配置一次.

## 配置 OpenWrt 上的防火墙

这里可以在 LuCI Web 管理界面中看到 服务 -> Tailscale 选项卡中, 点击 `自动配置防火墙`, 会添加 lan => tailscale 和 tailscale => lan 这些区域转发规则; 然而由于上面我们配置了 `通告出口节点`, 所以还需要添加以下规则

```
uci add firewall forwarding
uci set firewall.@forwarding[-1].src='tailscale'
uci set firewall.@forwarding[-1].dest='wan'
uci commit firewall
/etc/init.d/firewall restart
```
这样允许流量从 Tailscale 网络中通过节点访问公网, 实现流量出口代理.

## 其他注意事项
* 通常需要在 OpenWrt 上开启 IP forwarding, MASQUERADE 和 MSS 钳制, 以确保 Tailscale 网络中的流量可以正常转发和路由. 一般来说 Tailscale 客户端和 `自动配置防火墙` 会自动处理这些配置. 网络异常时可以检查这些配置.
* 内网网段如果冲突了, 可以在 OpenWrt 上修改对应的 DHCP 配置.
* 由于调整了较多的网络环境, 比如防火墙, 路由等, 就不建议再套用其他的网络应用, 比如去广告等功能, 可能会导致冲突.
