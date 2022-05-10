---
title: "[翻译] Kubernetes 网络模型指南"
date: 2019-10-28T18:55:42+08:00
draft: true
toc: false
images:
tags: 
  - Kubernetes
  - Translation
---

Kubernetes 旨在在一组机器上运行分布式系统。分布式系统的本质使网络成为 Kubernetes 部署中的核心且必要的部分，了解 Kubernetes 网络模型将使你能够正确地运行，监控和排查在 Kubernetes 上运行的应用程序。

网络是一个拥有许多成熟技术的广阔领域。对于不那么熟悉这个领域的人来说，这可能会令人感到不适，因为大多数人已经对网络有了先入为主的观念，而在 Kubernetes 中有很多新旧概念需要理解并将它们融合为一个整体。一个粗略的列表可能包括诸如网络命名空间，虚拟接口，IP 转发和网络地址转换之类的技术。本指南旨在通过讨论各种 Kubernetes 依靠的技术以及对这些技术是如何用来实现 Kubernetes 网络模型的描述来讲解 Kubernetes 网络模型。

本指南相当长，分为几个部分。我们首先讨论一些基本的 Kubernetes 术语以确保在整个指南中正确使用术语，然后讨论 Kubernetes 的网络模型以及它规定的设计和实现决策。接下来是本指南中最长且最有趣的部分：通过几种不同的用例深入讨论流量是如何在 Kubernetes 中被路由的。

如果您对一些网络术语，本指南附有网络术语词汇表。

## 目录

* 1 Kubernetes 基础知识
* 2 Kubernetes 网络模型
* 3 Container 到 Container 网络
* 4 Pod 到 Pod 网络
* 5 Pod 到 Service 网络
* 6 Internet 到 Service 网络
* 6.1 Egress
* 6.2 Ingress
* 7 总结
* 8 术语表

## 1 Kubernetes 基础知识

Kubernetes 是由一些核心概念构建而成的，这些核心概念被组合成越来越强大的功能。本节列出了这些概念，并提供了简要概述以帮助促进讨论。Kubernetes 的功能远不止这里列出的内容，本节作为入门使读者可以在后续部分中参考。如果您已经熟悉 Kubernetes，请随意跳过本节。

### 1.1 Kubernetes API 服务器

在 Kubernetes 中，一切都是由 Kubernetes API 服务器 (`kube-apiserver`) 提供的 API 调用。API 服务器是通往维护着应用程序集群的所需状态的 [etcd](https://github.com/coreos/etcd) 数据存储的网关。要更新 Kubernetes 集群的状态，您需要对 API 服务器进行 API 调用以描述所需的状态。

### 1.2 Controllers

Controller 是用于构建 Kubernetes 的核心抽象概念之一。在使用 API 服务器声明集群的期望状态后，Controller 将通过持续监视 API 服务器的状态并对所以变化做出反应来确保集群的当前状态与期望状态相匹配。Controller 使用一个简单的循环进行操作，该循环不断地根据群集当前状态对比群集的期望状态。如果存在任何差异，则控制器执行任务以确保当前状态与期望状态符合。用伪代码来表示：

```
while true:
  X = currentState()
  Y = desiredState()

  if X == Y:
    return  # Do nothing
  else:
    do(tasks to get to Y)
```

例如，当您使用 API 服务器创建新的 Pod 时，`kube-scheduler` (一个 Controller) 会注意到更改，并决定 Pod 部署在群集中的位置。然后，它使用 API 服务器 (由etcd支持) 写入状态变更。然后，`kubelet` (另一个 Controller) 会注意到新的更改，并建立所需的网络功能以使 Pod 在群集中可达。在这里，两个不同的控制器对两个不同的状态更改做出反应，以使集群的实际状况与用户的意图相匹配。

### 1.3 Pods

Pod 相当于 Kubernetes 中的原子 - 用于构建应用程序的最小可部署对象。单个 Pod 代表集群中正在运行的工作负载，并封装一个或多个 Docker 容器，任何必需的存储以及一个唯一的 IP 地址。组成 Pod 的容器在设计上是协同的，并在同一机器上进行调度。

### 1.4 Nodes

Node 是运行 Kubernetes 集群的机器。它们可以是裸机，虚拟机或其他东西。主机 (Host) 一词通常与 Node 交替使用。我会尽量使用一致的术语 Node，但有时会根据上下文使用虚拟机 (Virtual Machine) 一词来指代节点。

## 2 Kubernetes 网络模型

Kubernetes 对 Pods 的联网方式做出了固执的选择。特别的，Kubernetes 对所有的网络实现都作出了以下要求：

* 所有 Pod 都可以与所有其他 Pod 通信而无需使用网络地址转换 (NAT)
* 所有 Node 都可以在没有 NAT 的情况下与所有 Pod 通信
* Pod 所见的它的 IP 就是其他 Pod 看到的它的 IP

由于这些约束，我们现在有四个不同的联网问题需要解决：

1. Container 到 Container 网络
2. Pod 到 Pod 网络
3. Pod 到 Service 网络
3. Internet 到 Service 网络

指南剩下的部分将会依次讨论这些问题及其解决方法。

## 3 Container 到 Container 网络

通常，我们将虚拟机中的网络通信视为直接与以太网设备进行交互，如图1所示。

![Figure 1](/static/images/A-Guide-To-The-Kubernetes-Networking-Model/eth0.png)
图1. 以太网设备的理想状况

在现实中，情况更加微妙。在 Linux，每个运行中的进程通过 [网络命名空间](http://man7.org/linux/man-pages/man8/ip-netns.8.html) 进行通信，网络命名空间拥有它自己的路由规则、防火墙规则和网络设备。本质上，一个网络命名空间为命名空间内的所有进程提供了一个全新的网络栈。

Linux 用户可以用 `ip` 命令创造网络命名空间。例如，下面的命令创建了一个新的网络命名空间 `ns1`。

```
$ ip netns add ns1
```

当命名空间被创建时，它的挂载点 `/var/run/netns` 随之创建。这样即使没有进程归属于它也可以持久化。

你可以通过列出 `/var/run/netns` 挂载点来列出可用的命名空间，或者用 `ip` 命令。

```
$ ls /var/run/netns
ns1
$ ip netns
ns1
```

默认的，Linux 将所有进程分配给 root 网络命名空间来提供外部访问，如图2。

![Figure 2](/static/images/A-Guide-To-The-Kubernetes-Networking-Model/root-namespace.png)
图2. Root 网络命名空间

对于 Docker 的结构来说，一个 Pod 是一组共享同一网络命名空间的 Docker 容器。同一 Pod 中的容器拥有网络命名空间分配给 Pod 的相同的 IP 地址和端口范围，并且可以通过 localhost 找到各自，因为他们在同一命名空间内。我们为虚拟机中的每一个 Pod 创建网络命名空间。在 Docker 的实现中，一个 "Pod 容器" 打开网络命名空间，然后用户指定的 "app containers" 通过 Docker 的 –net=container: 功能加入这个网络命名空间。图3展示了多个容器 (`ctr*`) 组成的 Pod 在共享的命名空间的情景。

![Figure 3](/static/images/A-Guide-To-The-Kubernetes-Networking-Model/pod-namespace.png)
图3. 每个 Pod 一个网络命名空间

同一个 Pod 中的应用程序同样拥有共享卷的访问权，根据 Pod 的定义, 共享卷可挂载到每个应用程序的文件系统。

## 4 Pod 到 Pod 网络

在 Kubernetes 中，每个 Pod 都拥有真正的 IP 地址而且可以通过这个 IP 地址与别的 Pod 通信。当前需要理解 Kubernetes 是如何使 Pod 到 Pod 通信通过真正的 IP，无论 Pod 是在同一台物理 Node 上还是集群中的不同 Node 上。我们从假设 Pod 在相同的机器上开始讨论，避免 Node 之间通信在内部网络之外的复杂情况。

对于 Pod 来说，它在其命名空间中尝试去和在同一 Node 上不同网络命名空间通信。幸运的，命名空间可以通过 Linux [Virtual Ethernet Device](http://man7.org/linux/man-pages/man4/veth.4.html)，或通过由两个虚拟接口组成的可以扩展到多个网络命名空间的 `veth pair` 连接。为了连接 Pod 的命名空间，我们可以把 veth pair 的一端连接到 root 网络命名空间，另一端连接到 Pod 的网络命名空间。Veth pair 像跳线一样工作，连接两端使流量可以流通。这一步可以重复直到其数目和机器上的 Pods 一样多。图4展示了 veth pair 连接了所有 Pod 到 VM 的 root 命名空间的情况。

![Figure 4](/static/images/A-Guide-To-The-Kubernetes-Networking-Model/pod-veth-pairs.png)
图4. 每个 Pod 均有 veth pair

现在，我们建立了有独立网络命名空间的 Pod，让他们相信他们拥有他们自己的以太网设备和 IP 地址，然后他们连接到了 Node 的 root 命名空间上。现在，我们想让 Pod 通过 root 命名空间和彼此通信，为此我们使用网桥。

Linux 以太网网桥是一个用来连接两个或以上网段的虚拟的2层网络设备，透明地工作在两个网络上使其相连。网桥维护一个来源和目标的转发表，通过检测经过它的数据包的目的地来决定它是否将数据包传递给别的连接到桥上的网段。网桥码通过查找网络中以太网设备唯一的 MAC 地址来决定了是否桥接数据或将其丢弃。

网桥实现了 ARP 协议来发现链路层 MAC 地址关联的 IP 地址的。当数据帧被网桥接收时，网桥向所有连接的设备(除了原发送者)广播帧，响应的设备被存入查询表。之后带有相同 IP 地址的流量将会使用查询表来查找转发的正确 MAC 地址。

![Figure 5](/static/images/A-Guide-To-The-Kubernetes-Networking-Model/pods-connected-by-bridge.png)
图5. 通过网桥连接命名空间

### 4.1 包的生命周期：同一 Node 上 Pod 到 Pod

已有将 Pod 隔离到其网络堆栈的网络名称空间，将每个命名空间连接到 root 命名空间的虚拟以太网设备，和将命名空间连接在一起的网桥，我们终于准备好在同一 Node 上的 Pod 之间发送流量了。图6演示了这个过程。


![Figure 6](/static/images/A-Guide-To-The-Kubernetes-Networking-Model/pod-to-pod-same-node.gif)
图6. 包在同一个 Node 上 Pod 之间传递

在图6中，容器1将数据包发送到其自己的以太网设备 `eth0`，该设备作为容器的默认设备。对于 Pod1，`eth0` 通过虚拟以太网设备连接到 root 命名空间 `veth0` (1)。配置网桥 `cbr0` 连接到网段 `veth0`。数据包到达网桥后，网桥使用 ARP 协议将其解析到其正确的目标网段 `veth1` (3)。当数据包到达虚拟设备 `veth1` 时，它将直接被转发到 Pod2 的命名空间和该命名空间中的 `eth0` 设备 (4)。在整个流量流中，每个 Pod 仅与 `localhost` 上的 `eth0` 通信，并且流量被路由到正确的 Pod。对于这个网络的使用，我们的经验是它符合开发人员期望的默认行为。

Kubernetes的网络模型要求 Pod 必须通过其在跨 Node 中的 IP 地址才能访问。也就是说，一个 Pod 的 IP 地址始终对网络中的其他 Pod 可见，并且每个 Pod 所见的自己的 IP 地址都与其他 Pod 所见一致。现在，我们转向在不同 Node 上的 Pod 之间路由流量的问题。

### 4.2 包的生命周期：不同 Node 上 Pod 到 Pod

在确定了如何在同一 Node 上的 Pod 之间路由数据包之后，我们转向在不同 Node 上的 Pod 之间路由流量。Kubernetes 网络模型要求 Pod IP 在整个网络上可达，但是它没有指定必须如何完成。实际上，这是基于特定于网络的，但是已有一些模式被建立起来以简化此过程。

通常，群集中的每个节点都分配有一个 CIDR 块，该块指定了该 Node 上运行的 Pod 可用的 IP 地址。一旦发往 CID R块的流量到达 Node，则 Node 有责任将流量转发到正确的 Pod。图7说明了两个 Node 之间的流量流，假设网络可以将 CIDR 块中的流量路由到正确的 Node。

![Figure 7](/static/images/A-Guide-To-The-Kubernetes-Networking-Model/pod-to-pod-different-nodes.gif)
图7. 包在不同 Node 上的 Pod 之间传递

图7从与图6相同的请求开始，除了目标 Pod (以绿色突出显示)与源 Pod (以蓝色突出显示)位于不同的 Node 上。数据包首先通过 Pod1 的以太网设备发送，该设备与 root 命名空间中的虚拟以太网设备配对 (1)。数据包最终到达 root 名称空间的网桥 (2)。 ARP 将在网桥上失败，因为没有任何匹配数据包 MAC 地址的设备连接到网桥。网桥在失败时会将数据包送出默认路由 - root 命名空间 `eth0` 设备。此时，路由离开节点并进入网络 (3)。现在我们假设网络可以根据分配给 Node 的 CIDR 块将数据包路由到正确的 Node (4)。数据包进入目标 Node 的 root 命名空间 (VM 2上的 `eth0`)，然后通过网桥路由到正确的虚拟以太网设备 (5)。最后，通过流经 Pod4 命名空间中的虚拟以太网设备对来完成路由 (6)。一般而言，每个 Node 都知道如何将数据包传递到其中运行的 Pod。数据包到达目标 Node 后，数据包的流动方式与在同一 Node 上的 Pod 之间路由流量的方式相同。

我们省略了一步，即如何配置网络以将 Pod IP 的流量路由到负责这些 IP 的正确的 Node。这是特定于网络的，不过通过研究相关特例也可以提供其所涉及问题的一些见解。例如，对于 AWS，Amazon 为 Kubernetes 维护了一个容器网络插件，该插件通过 [容器网络接口(CNI)插件](https://github.com/aws/amazon-vpc-cni-k8s) 使在 Amazon VPC 环境中的 Node 到 Node 网络成为可能。

容器网络接口(CNI)提供了用于将容器连接到外部网络的通用 API。作为开发人员，我们想知道 Pod 可以使用 IP 地址与网络通信，并且我们希望此操作的机制透明。由 AWS 开发的 CNI 插件试图满足这些需求，同时通过 AWS 提供的现有 VPC，IAM 和安全组功能提供安全且可管理的环境。解决方案是使用弹性网络接口.

在 EC2 中，每个实例都绑定到一个弹性网络接口(ENI)，并且所有 ENI 都连接在 VPC 内 - ENI 可以相互访问，而无需付出额外的代价。默认情况下，每个 EC2 实例都部署有一个 ENI，但是您可以随意创建多个 ENI 并将它们部署到您认为合适的 EC2 实例中。用于 Kubernetes 的 AWS CNI 插件利用这种灵活性，为部署到 Node 的每一个 Pod 创建一个新的 ENI。由于 VPC 中的 ENI 已经连接到了现有的 AWS 基础设施，因此，每个在 VPC 中 Pod 的 IP 地址都是本地可寻址的。将 CNI 插件部署到群集后，每个 Node (EC2实例) 都会创建多个弹性网络接口，并为这些实例分配 IP 地址，从而为每个 Node 形成一个 CIDR 块。部署 Pod 时，作为 DaemonSet 部署到 Kubernetes 集群的小型二进制文件会收到所有的来自 Node 本地 kubelet 进程的将 Pod 添加到网络的请求。该二进制文件从 Node 的可用 ENI 池中选择一个可用 IP 地址，并通过在 Linux 内核中连接虚拟以太网设备和网桥，将其分配给 Pod，如在同一节点内将 Pod 联网时所述。有了这个，Pod 流量就可以在集群中跨 Node 进行路由。

## 5 Pod 到 Service 网络

我们已经展示了如何在 Pod 及其关联的 IP 地址之间路由流量。这非常有效，直到我们需要应对变化。 Pod IP 地址不是持久性的，并且会出现和消失，以应对规模扩大和缩小，应用程序的崩溃或节点重新启动。每一个这些事件都可以使 Pod IP 地址更改而不会发出任何警告。内置在 Kubernetes 的 Service 用以解决此问题。

Kubernetes 的 Service 管理一组 Pod 的状态，使您可以跟踪随时动态变化的一组 Pod IP 地址。Service 充当 Pod 的抽象，并为一组 Pod IP 地址分配一个虚拟 IP 地址。寻址到 Service 的虚拟 IP 的任何流量都将被路由到与虚拟 IP 关联的 Pod 集。这允许与 Service 关联的 Pod 集随时更改 - 客户端只需要了解 Service 的不变的虚拟 IP。

创建新的 Kubernetes Service 时，将为您创建一个新的虚拟IP (也称为群集IP)。在群集中的任何位置，寻址到虚拟 IP 的流量将负载均衡到与该 Service 关联的一组后端 Pod。实际上，Kubernetes 会自动创建并维护一个集群内的分布式负载均衡器，该负载均衡器会将流量分发到与 Service 相关联的健康 Pod。让我们仔细看看它是如何工作的。

5.1 netfilter 和 iptables

为了在群集内执行负载平衡，Kubernetes 依赖于 Linux 内置的网络框架 `netfilter`。Netfilter 是一个 Linux 提供的网络框架，允许通过定制的 handler 来实现一系列网络相关的操作。 Netfilter 提供了用于数据包过滤，网络地址转换和端口映射的各种功能和操作，这些功能和操作满足了网络中数据包重定向所需，并提供了禁止数据包到达计算机网络内敏感位置的功能。

Iptables 是一个用户空间程序，它提供一个基于表的系统，用于定义使用 netfilter 框架操作和转换数据包的规则。在Kubernetes 中，iptables 规则由监视 Kubernetes API 服务器更改的 kube-proxy Controller 配置。当对 Service 或 Pod 的更改更新了 Service 的虚拟 IP 地址或Pod 的 IP 地址时，将更新 iptables 规则以将针对 Service 的流量正确路由到后端 Pod。 Iptables 规则监视发往 Service的虚拟 IP 的流量，并在匹配项中从可用 Pod 的集合中选择一个随机的 Pod IP 地址，并且 iptables 规则将数据包的目标 IP 地址从服务的虚拟 IP 更改为 选中的 Pod 的 IP。随着 Pod 的规模扩大和缩小，iptables 规则集将更新以反映集群的变化状态。换句话说，iptables 在计算机上已通过将定向到 Service IP 的流量定向到实际 Pod 的 IP 完成了负载均衡，以。

在反过来的路径上，IP 地址来自目标 Pod。在这种情况下，iptables 再次重写 IP 头，用 Servicce 的 IP 替换 Pod 的 IP，使 Pod 认为它一直在与 Service 的 IP 进行通信。

### 5.2 IPVS

Kubernetes 的最新版本(1.11)包括集群负载均衡的第二个选项：IPVS。 IPVS (IP虚拟服务器) 也建立在 netfilter 之上，并作为 Linux 内核的一部分实现传输层负载均衡。 IPVS 已集成到 LVS (Linux 虚拟服务器) 中，在主机上运行，并充当真实服务器集群前面的负载均衡器。 IPVS 可以将基于 TCP 和 UDP 的服务的请求定向到真实服务器，并使由多台服务器组成的服务表现为在单个 IP 地址上的虚拟服务。这使得 IPVS 非常适合 Kubernetes Services。

声明 Kubernetes Service 时，可以指定是否要使用 iptables 或 IPVS 完成集群负载均衡。 IPVS 专为负载均衡而设计，并使用更有效的数据结构 (哈希表)，与 iptables 相比，几乎可以无限扩展。创建使用 IPVS 负载均衡的 Service 时，会发生三件事：在 Node 上创建一个虚拟 IPVS 接口，将 Service 的 IP 地址绑定到该虚拟 IPVS 接口，并为每个 Service IP 地址创建 IPVS 服务器。

将来，IPVS 有望成为群集内负载均衡的默认方法。这个更改仅影响群集内负载均衡，在本指南的其余部分中，您可以使用 IPVS 安全地将 iptables 替换为群集内负载平衡，而不会影响其余的讨论。现在，让我们看一下通过群集内负载平衡 Service 的数据包的生命周期。

### 5.3 包的生命周期：Pod 到 Service

![Figure 8](/static/images/A-Guide-To-The-Kubernetes-Networking-Model/pod-to-service.gif)
Figure 8. 数据包在 Pod 和 Service 间移动

When routing a packet between a Pod and Service, the journey begins in the same way as before. The packet first leaves the Pod through the eth0 interface attached to the Pod’s network namespace (1). Then it travels through the virtual Ethernet device to the bridge (2). The ARP protocol running on the bridge does not know about the Service and so it transfers the packet out through the default route — eth0 (3). Here, something different happens. Before being accepted at eth0, the packet is filtered through iptables. After receiving the packet, iptables uses the rules installed on the Node by kube-proxy in response to Service or Pod events to rewrite the destination of the packet from the Service IP to a specific Pod IP (4). The packet is now destined to reach Pod 4 rather than the Service’s virtual IP. The Linux kernel’s conntrack utility is leveraged by iptables to remember the Pod choice that was made so future traffic is routed to the same Pod (barring any scaling events). In essence, iptables has done in-cluster load balancing directly on the Node. Traffic then flows to the Pod using the Pod-to-Pod routing we’ve already examined (5).

在Pod和Service之间路由数据包时，旅程以与以前相同的方式开始。数据包首先通过附加到Pod网络名称空间（1）的eth0接口离开Pod。然后，它通过虚拟以太网设备到达网桥（2）。在网桥上运行的ARP协议不了解服务，因此它通过默认路由eth0（3）传送数据包。在这里，发生了一些不同的事情。在被eth0接受之前，该数据包已通过iptables进行过滤。收到数据包后，iptables会使用kube代理安装在节点上的规则来响应Service或Pod事件，将数据包的目标从Service IP重写到特定的Pod IP（4）。现在，该数据包将到达Pod 4，而不是服务的虚拟IP。 iptables充分利用了Linux内核的conntrack实用程序，以记住做出的Pod选择，以便将来的流量被路由到相同的Pod（除非有任何扩展事件）。本质上，iptables直接在节点上完成了集群负载平衡。然后，使用我们已经检查过的Pod到Pod路由，流量流向Pod。

5.4 Life of a packet: Service to Pod
service-to-pod.gif
Figure 9. Packets moving between Services and Pods.

The Pod that receives this packet will respond, identifying the source IP as its own and the destination IP as the Pod that originally sent the packet (1). Upon entry into the Node, the packet flows through iptables, which uses conntrack to remember the choice it previously made and rewrite the source of the packet to be the Service’s IP instead of the Pod’s IP (2). From here, the packet flows through the bridge to the virtual Ethernet device paired with the Pod’s namespace (3), and to the Pod’s Ethernet device as we’ve seen before (4).

接收到此数据包的Pod将做出响应，将源IP标识为自己的IP，将目标IP标识为最初发送该数据包的Pod（1）。进入节点后，数据包流经iptables，后者使用conntrack记住其先前所做的选择，并将数据包的源重写为服务的IP，而不是Pod的IP（2）。数据包从这里流过网桥，到达与Pod的命名空间配对的虚拟以太网设备（3），再到我们之前所见的Pod的以太网设备（4）。

5.5 Using DNS

Kubernetes can optionally use DNS to avoid having to hard-code a Service’s cluster IP address into your application. Kubernetes DNS runs as a regular Kubernetes Service that is scheduled on the cluster. It configures the kubelets running on each Node so that containers use the DNS Service’s IP to resolve DNS names. Every Service defined in the cluster (including the DNS server itself) is assigned a DNS name. DNS records resolve DNS names to the cluster IP of the Service or the IP of a POD, depending on your needs. SRV records are used to specify particular named ports for running Services.

Kubernetes可以选择使用DNS，以避免必须将服务的群集IP地址硬编码到您的应用程序中。 Kubernetes DNS作为在群集上计划的常规Kubernetes服务运行。它配置在每个节点上运行的kubelet，以便容器使用DNS服务的IP来解析DNS名称。为群集中定义的每个服务（包括DNS服务器本身）分配一个DNS名称。 DNS记录根据您的需要将DNS名称解析为服务的群集IP或POD的IP。 SRV记录用于指定运行服务的特定命名端口。

A DNS Pod consists of three separate containers:

kubedns: watches the Kubernetes master for changes in Services and Endpoints, and maintains in-memory lookup structures to serve DNS requests.
dnsmasq: adds DNS caching to improve performance.
sidecar: provides a single health check endpoint to perform healthchecks for dnsmasq and kubedns.

kubedns：监视Kubernetes主服务器的服务和端点更改，并维护内存查找结构以服务DNS请求。 
dnsmasq：添加DNS缓存以提高性能。 
sidecar：提供单个运行状况检查端点，以执行dnsmasq和kubedns的运行状况检查。

The DNS Pod itself is exposed as a Kubernetes Service with a static cluster IP that is passed to each running container at startup so that each container can resolve DNS entries. DNS entries are resolved through the kubedns system that maintains in-memory DNS representations. etcd is the backend storage system for cluster state, and kubedns uses a library that converts etcd key-value stores to DNS entires to rebuild the state of the in-memory DNS lookup structure when necessary.

DNS Pod本身作为Kubernetes服务公开，具有静态群集IP，该IP在启动时传递给每个正在运行的容器，以便每个容器都可以解析DNS条目。 DNS条目通过kubedns系统解析，该系统在内存中维护DNS表示形式。 etcd是用于群集状态的后端存储系统，而kubedns使用一个库，该库在必要时将etcd密钥值存储转换为DNS整体，以重建内存DNS查找结构的状态。

CoreDNS works similarly to kubedns but is built with a plugin architecture that makes it more flexible. As of Kubernetes 1.11, CoreDNS is the default DNS implementation for Kubernetes.

CoreDNS的工作方式与kubedns相似，但其使用的插件体系结构使其更加灵活。从Kubernetes 1.11开始，CoreDNS是Kubernetes的默认DNS实现。

6 Internet-to-Service Networking

So far we have looked at how traffic is routed within a Kubernetes cluster. This is all fine and good but unfortunately walling off your application from the outside world will not help meet any sales targets — at some point you will want to expose your service to external traffic. This need highlights two related concerns: (1) getting traffic from a Kubernetes Service out to the Internet, and (2) getting traffic from the Internet to your Kubernetes Service. This section deals with each of these concerns in turn.

到目前为止，我们已经研究了如何在Kubernetes集群中路由流量。一切都很好，但是不幸的是，与外界隔离您的应用程序将无助于达到任何销售目标-在某个时候，您将需要向外部流量公开您的服务。这种需求突出了两个相关的问题：（1）将来自Kubernetes服务的流量发送到Internet，以及（2）将来自Internet的流量发送到您的Kubernetes Service。本节依次阐述这些问题。

6.1 Egress — Routing traffic to the Internet

Routing traffic from a Node to the public Internet is network specific and really depends on how your network is configured to publish traffic. To make this section more concrete I will use an AWS VPC to discuss any specific details.

从节点到公共Internet的流量路由是特定于网络的，并且实际上取决于网络配置为发布流量的方式。为了使本节更具体，我将使用AWS VPC讨论任何特定细节。

In AWS, a Kubernetes cluster runs within a VPC, where every Node is assigned a private IP address that is accessible from within the Kubernetes cluster. To make traffic accessible from outside the cluster, you attach an Internet gateway to your VPC. The Internet gateway serves two purposes: providing a target in your VPC route tables for traffic that can be routed to the Internet, and performing network address translation (NAT) for any instances that have been assigned public IP addresses. The NAT translation is responsible for changing the Node’s internal IP address that is private to the cluster to an external IP address that is available in the public Internet.

在AWS中，Kubernetes集群在VPC内运行，其中为每个节点分配了一个私有IP地址，该地址可从Kubernetes集群内访问。要使流量可以从群集外部访问，请将Internet网关连接到VPC。 Internet网关有两个目的：在VPC路由表中为可路由到Internet的流量提供目标，并为已分配了公共IP地址的任何实例执行网络地址转换（NAT）。NAT转换负责将群集专用的节点内部IP地址更改为公用Internet上可用的外部IP地址。

With an Internet gateway in place, VMs are free to route traffic to the Internet. Unfortunately, there is a small problem. Pods have their own IP address that is not the same as the IP address of the Node that hosts the Pod, and the NAT translation at the Internet gateway only works with VM IP addresses because it does not have any knowledge about what Pods are running on which VMs — the gateway is not container aware. Let’s look at how Kubernetes solves this problem using iptables (again).

有了Internet网关后，VM可以自由地将流量路由到Internet。不幸的是，这是一个小问题。 Pod拥有自己的IP地址，该IP地址与托管Pod的节点的IP地址不同，并且Internet网关上的NAT转换仅适用于VM IP地址，因为它不了解运行Pod的任何知识哪些虚拟机-网关不支持容器。让我们看看Kubernetes如何使用iptables解决这个问题（再次）。

6.1.1 Life of a packet: Node to Internet

In the following diagram, the packet originates at the Pod’s namespace (1) and travels through the veth pair connected to the root namespace (2). Once in the root namespace, the packet moves from the bridge to the default device since the IP on the packet does not match any network segment connected to the bridge. Before reaching the root namespace’s Ethernet device (3), iptables mangles the packet (3). In this case, the source IP address of the packet is a Pod, and if we keep the source as a Pod the Internet gateway will reject it because the gateway NAT only understands IP addresses that are connected to VMs. The solution is to have iptables perform a source NAT — changing the packet source — so that the packet appears to be coming from the VM and not the Pod. With the correct source IP in place, the packet can now leave the VM (4) and reach the Internet gateway (5). The Internet gateway will do another NAT rewriting the source IP from a VM internal IP to an external IP. Finally, the packet will reach the public Internet (6). On the way back, the packet follows the same path and any source IP mangling is undone so that each layer of the system receives the IP address that it understands: VM-internal at the Node or VM level, and a Pod IP within a Pod’s namespace.

在下图中，数据包起源于Pod的名称空间（1），并经过与根名称空间（2）连接的第ve对。一旦进入根名称空间，数据包就会从网桥移动到默认设备，因为数据包上的IP与连接到网桥的任何网段都不匹配。在到达根名称空间的以太网设备（3）之前，iptables会处理数据包（3）。在这种情况下，数据包的源IP地址是Pod，如果我们将源保留为Pod，则Internet网关将拒绝它，因为网关NAT仅了解连接到VM的IP地址。解决方案是让iptables执行源NAT（更改数据包源），以便数据包看起来是来自VM而不是Pod。有了正确的源IP，数据包现在可以离开VM（4）并到达Internet网关（5）。 Internet网关将执行另一个NAT，将源IP从VM内部IP重写为外部IP。最终，数据包将到达公共Internet（6）。在返回的过程中，数据包遵循相同的路径，并且所有源IP处理都被撤消，因此系统的每一层都接收其能够理解的IP地址：节点或VM级别的内部VM，以及Pod命名空间中的Pod IP 。

pod-to-internet.gif
Figure 10. Routing packets from Pods to the Internet.

6.2 Ingress — Routing Internet traffic to Kubernetes

Ingress — getting traffic into your cluster — is a surprisingly tricky problem to solve. Again, this is specific to the network you are running, but in general, Ingress is divided into two solutions that work on different parts of the network stack: (1) a Service LoadBalancer and (2) an Ingress controller.

入口（将流量引入群集）是一个非常棘手的问题。同样，这是特定于您正在运行的网络的，但是通常，Ingress分为两个可在网络堆栈的不同部分上运行的解决方案：（1）Service LoadBalancer和（2）Ingress控制器。

6.2.1 Layer 4 Ingress: LoadBalancer

When you create a Kubernetes Service you can optionally specify a LoadBalancer to go with it. The implementation of the LoadBalancer is provided by a cloud controller that knows how to create a load balancer for your service. Once your Service is created, it will advertise the IP address for the load balancer. As an end user, you can start directing traffic to the load balancer to begin communicating with your Service.

创建Kubernetes服务时，可以选择指定一个LoadBalancer来配合它。云控制器提供了LoadBalancer的实现，该控制器知道如何为您的服务创建负载平衡器。创建服务后，它将为负载平衡器通告IP地址。作为最终用户，您可以开始将流量定向到负载平衡器，以开始与服务进行通信。

With AWS, load balancers are aware of Nodes within their Target Group and will balance traffic throughout all of the Nodes in the cluster. Once traffic reaches a Node, the iptables rules previously installed throughout the cluster for your Service will ensure that traffic reaches the Pods for the Service you are interested in.

借助AWS，负载平衡器可以知道其目标组中的节点，并将平衡群集中所有节点上的流量。一旦流量到达节点，先前在整个群集中为您的服务安装的iptables规则将确保流量到达您感兴趣的服务的Pod。

6.2.2 Life of a packet: LoadBalancer to Service

Let’s look at how this works in practice. Once you deploy your service, a new load balancer will be created for you by the cloud provider you are working with (1). Because the load balancer is not container aware, once traffic reaches the load-balancer it is distributed throughout the VMs that make up your cluster (2). iptables rules on each VM will direct incoming traffic from the load balancer to the correct Pod (3) — these are the same IP tables rules that were put in place during Service creation and discussed earlier. The response from the Pod to the client will return with the Pod’s IP, but the client needs to have the load balancer’s IP address. iptables and conntrack is used to rewrite the IPs correctly on the return path, as we saw earlier.

让我们看看这在实践中是如何工作的。部署服务后，您正在使用的云提供商将为您创建一个新的负载均衡器（1）。由于负载平衡器不支持容器，因此，一旦流量到达负载平衡器，流量便会分布在组成群集的所有VM中（2）。每个VM上的iptables规则会将来自负载均衡器的传入流量定向到正确的Pod（3），这些规则与创建服务期间制定的IP表规则相同，前面已经讨论过。从Pod到客户端的响应将返回Pod的IP，但客户端需要具有负载均衡器的IP地址。如前所述，iptables和conntrack用于在返回路径上正确重写IP。

The following diagram shows a network load balancer in front of three VMs that host your Pods. Incoming traffic (1) is directed at the load balancer for your Service. Once the load balancer receives the packet (2) it picks a VM at random. In this case, we’ve chosen pathologically the VM with no Pod running: VM 2 (3). Here, the iptables rules running on the VM will direct the packet to the correct Pod using the internal load balancing rules installed into the cluster using kube-proxy. iptables does the correct NAT and forwards the packet on to the correct Pod (4).

下图显示了在承载Pod的三个VM之前的网络负载平衡器。传入流量（1）指向服务的负载平衡器。一旦负载均衡器收到数据包（2），它就会随机选择一个VM。在这种情况下，我们从病理上选择了没有运行Pod的VM：VM 2（3）。在这里，在VM上运行的iptables规则将使用通过kube代理安装到群集中的内部负载平衡规则将数据包定向到正确的Pod。 iptables执行正确的NAT，并将数据包转发到正确的Pod（4）。

internet-to-service.gif
Figure 11. Packets sent from the Internet to a Service.
6.2.3 Layer 7 Ingress: Ingress Controller

Layer 7 network Ingress operates on the HTTP/HTTPS protocol range of the network stack and is built on top of Services. The first step to enabling Ingress is to open a port on your Service using the NodePort Service type in Kubernetes. If you set the Service’s type field to NodePort, the Kubernetes master will allocate a port from a range you specify, and each Node will proxy that port (the same port number on every Node) into your Service. That is, any traffic directed to the Node’s port will be forwarded on to the service using iptables rules. This Service to Pod routing follows the same internal cluster load-balancing pattern we’ve already discussed when routing traffic from Services to Pods.

第7层网络入口在网络堆栈的HTTP / HTTPS协议范围内运行，并建立在服务之上。启用Ingress的第一步是使用Kubernetes中的NodePort服务类型在服务上打开端口。如果将服务的类型字段设置为NodePort，则Kubernetes主服务器将在您指定的范围内分配端口，并且每个节点会将该端口（每个节点上的相同端口号）代理到您的服务中。也就是说，使用iptables规则，任何定向到该节点端口的流量都将转发到该服务。从服务到Pod的路由遵循了我们已经讨论过的相同的内部集群负载平衡模式。

To expose a Node’s port to the Internet you use an Ingress object. An Ingress is a higher-level HTTP load balancer that maps HTTP requests to Kubernetes Services. The Ingress method will be different depending on how it is implemented by the Kubernetes cloud provider controller. HTTP load balancers, like Layer 4 network load balancers, only understand Node IPs (not Pod IPs) so traffic routing similarly leverages the internal load-balancing provided by the iptables rules installed on each Node by kube-proxy.

要将节点的端口暴露给Internet，请使用Ingress对象。 Ingress是更高级别的HTTP负载平衡器，可将HTTP请求映射到Kubernetes Services。 Ingress方法将有所不同，具体取决于Kubernetes云提供商控制器如何实现。 HTTP负载平衡器（如第4层网络负载平衡器）仅了解节点IP（而非Pod IP），因此流量路由同样利用kube代理在每个节点上安装的iptables规则提供的内部负载平衡。

Within an AWS environment, the ALB Ingress Controller provides Kubernetes Ingress using Amazon’s Layer 7 Application Load Balancer. The following diagram details the AWS components this controller creates. It also demonstrates the route Ingress traffic takes from the ALB to the Kubernetes cluster.

在AWS环境中，ALB入口控制器使用Amazon的7层应用程序负载平衡器提供Kubernetes入口。下图详细说明了此控制器创建的AWS组件。它还演示了入口流量从ALB到Kubernetes集群的路线。

ingress-controller-design.png
Figure 12. Design of an Ingress Controller.

Upon creation, (1) an Ingress Controller watches for Ingress events from the Kubernetes API server. When it finds Ingress resources that satisfy its requirements, it begins the creation of AWS resources. AWS uses an Application Load Balancer (ALB) (2) for Ingress resources. Load balancers work in conjunction with Target Groups that are used to route requests to one or more registered Nodes. (3) Target Groups are created in AWS for each unique Kubernetes Service described by the Ingress resource. (4) A Listener is an ALB process that checks for connection requests using the protocol and port that you configure. Listeners are created by the Ingress controller for every port detailed in your Ingress resource annotations. Lastly, Target Group Rules are created for each path specified in your Ingress resource. This ensures traffic to a specific path is routed to the correct Kubernetes Service (5).

创建后，（1）入口控制器将监视来自Kubernetes API服务器的入口事件。当发现满足其要求的Ingress资源时，它将开始创建AWS资源。 AWS将应用程序负载平衡器（ALB）（2）用于Ingress资源。负载均衡器与用于将请求路由到一个或多个已注册节点的目标组配合使用。 （3）在AWS中为Ingress资源描述的每个唯一的Kubernetes服务创建目标组。（4）侦听器是ALB进程，它使用您配置的协议和端口检查连接请求。侦听器是由Ingress控制器为Ingress资源注释中详细说明的每个端口创建的。最后，为Ingress资源中指定的每个路径创建目标组规则。这样可以确保将到特定路径的流量路由到正确的Kubernetes服务（5）。

6.2.4 Life of a packet: Ingress to Service

The life of a packet flowing through an Ingress is very similar to that of a LoadBalancer. The key differences are that an Ingress is aware of the URL’s path (allowing and can route traffic to services based on their path), and that the initial connection between the Ingress and the Node is through the port exposed on the Node for each service.

流经Ingress的数据包的寿命与LoadBalancer的寿命非常相似。主要区别在于，Ingress知道URL的路径（允许并可以根据服务的路径将流量路由到服务），并且Ingress和Node之间的初始连接是通过Node上每个服务暴露的端口进行的。

Let’s look at how this works in practice. Once you deploy your service, a new Ingress load balancer will be created for you by the cloud provider you are working with (1). Because the load balancer is not container aware, once traffic reaches the load-balancer it is distributed throughout the VMs that make up your cluster (2) through the advertised port for your service. iptables rules on each VM will direct incoming traffic from the load balancer to the correct Pod (3) — as we have seen before. The response from the Pod to the client will return with the Pod’s IP, but the client needs to have the load balancer’s IP address. iptables and conntrack is used to rewrite the IPs correctly on the return path, as we saw earlier.

让我们看看这在实践中是如何工作的。部署服务后，您正在使用的云提供商将为您创建一个新的Ingress负载均衡器（1）。由于负载平衡器不支持容器，因此，一旦流量到达负载平衡器，它将通过为您的服务通告的端口在组成您的群集（2）的所有VM中进行分配。如前所述，每个VM上的iptables规则会将来自负载均衡器的传入流量定向到正确的Pod（3）。从Pod到客户端的响应将返回Pod的IP，但客户端需要具有负载均衡器的IP地址。如前所述，iptables和conntrack用于在返回路径上正确重写IP。

ingress-to-service.gif
Figure 13. Packets sent from an Ingress to a Service.

One benefit of Layer 7 load-balancers are that they are HTTP aware, so they know about URLs and paths. This allows you you to segment your Service traffic by URL path. They also typically provide the original client’s IP address in the X-Forwarded-For header of the HTTP request.

第7层负载平衡器的一个好处是它们可以识别HTTP，因此他们知道URL和路径。这使您可以按URL路径细分服务流量。它们通常还会在HTTP请求的X Forwarded For标头中提供原始客户端的IP地址。

7 Wrapping Up
This guide provides a foundation for understanding the Kubernetes networking model and how it enables common networking tasks. The field of networking is both broad and deep and it’s impossible to cover everything here. This guide should give you a starting point to dive into the topics you are interested in and want to know more about. Whenever you are stumped, leverage the Kubernetes documentation and the Kubernetes community to help you find your way.

8 Glossary

Kubernetes relies on several existing technologies to build a functioning cluster. Fully exploring each of these technologies is beyond the scope of this guide, but this section describes each of those technologies in enough detail to follow along with the discussion. You can feel free to skim this section, skip it completely, or refer to it as needed if you ever get confused or need a refresher.

Kubernetes依靠几种现有技术来构建可运行的集群。全面探索每种技术不在本指南的讨论范围内，但是本节将对每种技术进行足够详细的介绍，以供讨论时参考。您可以随意浏览本节，完全跳过本节，或者在感到困惑或需要复习时根据需要参考。

Layer 2 Networking
Layer 2 is the data link layer providing Node-to-Node data transfer. It defines the protocol to establish and terminate a connection between two physically connected devices. It also defines the protocol for flow control between them.

Layer 4 Networking
The transport layer controls the reliability of a given link through flow control. In TCP/IP, this layer refers to the TCP protocol for exchanging data over an unreliable network.

Layer 7 Networking
The application layer is the layer closest to the end user, which means both the application layer and the user interact directly with the software application. This layer interacts with software applications that implement a communicating component. Typically, Layer 7 Networking refers to HTTP.

NAT — Network Address Translation
NAT or network address translation is an IP-level remapping of one address space into another. The mapping happens by modifying network address information in the IP header of packets while they are in transit across a traffic routing device.

A basic NAT is a simple mapping from one IP address to another. More commonly, NAT is used to map multiple private IP address into one publicly exposed IP address. Typically, a local network uses a private IP address space and a router on that network is given a private address in that space. The router is then connected to the Internet with a public IP address. As traffic is passed from the local network to the Internet, the source address for each packet is translated from the private address to the public address, making it seem as though the request is coming directly from the router. The router maintains connection tracking to forward replies to the correct private IP on the local network.

NAT provides an additional benefit of allowing large private networks to connect to the Internet using a single public IP address, thereby conserving the number of publicly used IP addresses.

SNAT — Source Network Address Translation
SNAT simply refers to a NAT procedure that modifies the source address of an IP packet. This is the typical behaviour for the NAT described above.

DNAT — Destination Network Address Translation
DNAT refers to a NAT procedure that modifies the destination address of an IP packet. DNAT is used to publish a service resting in a private network to a publicly addressable IP address.

Network Namespace
In networking, each machine (real or virtual) has an Ethernet device (that we will refer to as eth0). All traffic flowing in and out of the machine is associated with that device. In truth, Linux associates each Ethernet device with a network namespace — a logical copy of the entire network stack, with its own routes, firewall rules, and network devices. Initially, all the processes share the same default network namespace from the init process, called the root namespace. By default, a process inherits its network namespace from its parent and so, if you don’t make any changes, all network traffic flows through the Ethernet device specified for the root network namespace.

veth — Virtual Ethernet Device Pairs
Computer systems typically consist of one or more networking devices — eth0, eth1, etc — that are associated with a physical network adapter which is responsible for placing packets onto the physical wire. Veth devices are virtual network devices that are always created in interconnected pairs. They can act as tunnels between network namespaces to create a bridge to a physical network device in another namespace, but can also be used as standalone network devices. You can think of a veth device as a virtual patch cable between devices — what goes in one end will come out the other.

bridge — Network Bridge
A network bridge is a device that creates a single aggregate network from multiple communication networks or network segments. Bridging connects two separate networks as if they were a single network. Bridging uses an internal data structure to record the location that each packet is sent to as a performance optimization.

CIDR — Classless Inter-Domain Routing
CIDR is a method for allocating IP addresses and performing IP routing. With CIDR, IP addresses consist of two groups: the network prefix (which identifies the whole network or subnet), and the host identifier (which specifies a particular interface of a host on that network or subnet). CIDR represents IP addresses using CIDR notation, in which an address or routing prefix is written with a suffix indicating the number of bits of the prefix, such as 192.0.2.0/24 for IPv4. An IP address is part of a CIDR block, and is said to belong to the CIDR block if the initial n bits of the address and the CIDR prefix are the same.

CNI — Container Network Interface
CNI (Container Network Interface) is a Cloud Native Computing Foundation project consisting of a specification and libraries for writing plugins to configure network interfaces in Linux containers. CNI concerns itself only with network connectivity of containers and removing allocated resources when the container is deleted.

VIP — Virtual IP Address
A virtual IP address, or VIP, is a software-defined IP address that doesn’t correspond to an actual physical network interface.

netfilter — The Packet Filtering Framework for Linux
netfilter is the packet filtering framework in Linux. The software implementing this framework is responsible for packet filtering, network address translation (NAT), and other packet mangling.

netfilter, ip_tables, connection tracking (ip_conntrack, nf_conntrack) and the NAT subsystem together build the major parts of the framework.

iptables — Packet Mangling Tool
iptables is a program that allows a Linux system administrator to configure the netfilter and the chains and rules it stores. Each rule within an IP table consists of a number of classifiers (iptables matches) and one connected action (iptables target).

conntrack — Connection Tracking
conntrack is a tool built on top of the Netfilter framework to handle connection tracking. Connection tracking allows the kernel to keep track of all logical network connections or sessions, and direct packets for each connection or session to the correct sender or receiver. NAT relies on this information to translate all related packets in the same way, and iptables can use this information to act as a stateful firewall.

IPVS — IP Virtual Server
IPVS implements transport-layer load balancing as part of the Linux kernel.

IPVS is a tool similar to iptables. It is based on the Linux kernel’s netfilter hook function, but uses a hash table as the underlying data structure. That means, when compared to iptables, IPVS redirects traffic much faster, has much better performance when syncing proxy rules, and provides more load balancing algorithms.

DNS — The Domain Name System
The Domain Name System (DNS) is a decentralized naming system for associating system names with IP addresses. It translates domain names to numerical IP addresses for locating computer services.

