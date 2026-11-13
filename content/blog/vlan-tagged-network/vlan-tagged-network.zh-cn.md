---
title: "Docker 容器的 VLAN 转发"
date: "2025-11-21"
tags: ["Network", "VLAN", "Docker"]
---

在一些工业或老旧的网络环境中，某些设备（比如 QNX 系统）要求流量带有特定的 VLAN 标签。而 Docker 容器默认不会直接发送带 VLAN 标签的流量。本文解释背后的原理，并介绍如何解决这个问题。

## 什么是 VLAN

VLAN（Virtual Local Area Network，虚拟局域网）是一种二层（[OSI 第二层](https://en.wikipedia.org/wiki/OSI_model#Layer_2:_Data_link_layer)）技术，它把一个物理网络划分成多个逻辑网络。每个 VLAN 有一个唯一的 ID（802.1Q 标签）。不同 VLAN 中的设备不能直接通信，除非经过路由。

### 为什么 Docker 不支持 VLAN

1. Docker 默认的网络模型是三层（基于 IP）的，而不是二层

VLAN（IEEE 802.1Q）是纯粹的二层技术。但 Docker 默认的网络驱动（bridge、overlay、host、macvlan、ipvlan）负责的是：

- 管理 IP 路由
- 分配 IP 地址
- 做 NAT、路由和转发
- 屏蔽二层帧的细节

Docker 隐藏或虚拟化了二层的细节，容器不会直接处理以太网帧。

2. Docker 的 "bridge" 网络使用的虚拟交换机会去掉 VLAN 标签

Docker bridge（docker0）是用 Linux bridge 实现的：除非显式开启 VLAN 过滤，否则 Linux bridge 会丢掉 VLAN 标签。
Docker 把 bridge 配置成简单的三层路由模式，不会配置 VLAN 感知，也不会做按端口的 VLAN 映射。所以容器发出的任何带 VLAN 标签的帧，都会在 bridge 上丢失。

3. VLAN 需要直接控制以太网帧，但 Docker 的驱动会过滤或封装这些帧

- bridge 驱动：在 Linux bridge 上去掉 VLAN 标签。
- overlay 驱动：把流量封装在 VXLAN 中，丢弃 VLAN 标签等所有二层信息。
- macvlan / ipvlan：可以传递二层流量，但只有宿主机能添加 VLAN 子接口，Docker 不能。

所以 Docker 从来没有提供让容器生成原始 802.1Q 帧的途径。即使使用 macvlan，VLAN 也必须在宿主机上配置，而不是在 Docker 里。

### 为什么 GRE 隧道（AWS）不支持 VLAN

AWS 的 GRE 实现（用于 TGW Connect、Direct Connect + GRE、SD-WAN 集成等）只作为三层 GRE 封装服务工作。由于这种架构设计，VLAN 标签（802.1Q 头）会被有意去掉或拒绝。

AWS GRE 隧道只承载：

- IPv4 包
- IPv6 包（取决于具体服务）

AWS 中的 GRE 帧不支持二层载荷，例如：

- 以太网帧
- MAC 地址
- 802.1Q VLAN 头
- ARP、LLDP、STP 等

VLAN 标签是以太网头的一部分，而 AWS 的 GRE 传输从不传递以太网帧，所以 VLAN 标签根本没有存在的位置。

## 如何解决

要让容器能和需要 VLAN 的设备通信，需要：

1. 在宿主机上创建 VLAN 子接口：

```bash
sudo ip link add link eth0 name eth0.100 type vlan id 100
sudo ip link set dev eth0.100 up
```

> [!NOTE]
> 确保防火墙和网络策略允许改写过 IP/MAC 的流量通过。

2. 添加一个 bridge，把原来不带标签的网卡（eth0）和带标签的网卡（eth0.100）连起来

```bash
sudo ip link add name br0 type bridge
sudo ip link set br0 up
sudo ip link set eth0 master br0
sudo ip link set eth0.100 master br0
```

这样，从 eth0 进来的流量（不带标签）会被桥接到 eth0.100，并加上 VLAN 100 标签。
从 eth0.100 进来的流量（带 VLAN 100 标签）会被桥接到 eth0，并去掉标签。

## 进阶建议

如果想把流量转发到外部设备，可以用 Linux Traffic Control（tc）改写整个二层/三层头（MAC、IP、TTL），从而把流量转发到物理网卡上。

例如：

```bash
tc filter add dev eth0.100 ingress protocol ip pref 10 flower \
  ip_proto udp src_ip "$SRC_IP" dst_ip "$DST_IP" \
  src_port "$SRC_PORT" dst_port "$DST_PORT" \
  action pedit ex munge ip src set "$SRC_IP_NEW" pipe \
  ...
```
