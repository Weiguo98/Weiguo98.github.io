---
title: "外部网络无法与 Docker 容器通信的解决方案"
date: "2025-09-05"
tags: ["Docker", "Network", "Troubleshoot"]
---

本文一步步介绍如何排查和解决外部网络无法与 Docker 容器通信的问题，尤其是 UDP 流量。

## 背景

用 `docker-compose` 启动一组使用自定义 bridge 网络（例如 `test-network`）的容器时，可能会遇到 UDP 包在容器之间、或容器与外部服务之间无法正常传输的问题。通过分析 `iptables`、`nftables` 和容器的监听状态，可以让容器在指定端口上正确监听并响应 UDP 包。

## 网络结构概览

用下面的命令查看 Docker 网络：

```bash
docker network inspect test-network
```

输出中可能包含这些关键配置：

```json
{
    "Subnet": "168.17.5.0/22",
    "Gateway": "168.17.7.253",
    "Options": {
        # 你想在本机连接的 bridge 网络名称
        "com.docker.network.bridge.name": "localbridge"
    }
}
```

该网络中的容器 IP 示例：

- 168.17.7.254
- 168.17.5.1
- 168.17.5.20

### 问题描述

这种场景下常见的问题有：

- 从某个容器发往外部 IP（例如 `168.19.1.1`）的 UDP 包收不到响应。
- ping 容器 IP 不通。
- 用 `tcpdump` 能看到包发出去了，但没有回复。
- 抓包日志中出现类似错误：udp port 0030 unreachable

## 排查步骤

### 1. 检查容器监听的端口

在容器内运行下面的命令，查看正在监听哪些端口：

```bash
netstat -uln
```

如果输出类似：

```
127.0.0.11:34321
```

说明容器内的服务没有在实际的业务端口（例如 `0030`）上监听，或者没有监听所有网卡。

### 2. 修改容器服务，监听正确的端口

确保容器内的服务监听在正确的地址和端口上，例如：

- `0.0.0.0:0030`（所有网卡）
- 或者指定 IP，例如 `168.17.5.x:0030`

可以用下面的命令确认：

```bash
ss -uln
```

### 3. 在 `docker-compose` 中开放 UDP 端口

#### 容器之间通信

在 `docker-compose.yml` 中使用 `expose`：

```yaml {title="docker-compose.yml"}
services:
  app:
    image: your_image
    expose:
      - "0030/udp"
```

#### 宿主机与容器通信

使用 `ports` 把容器端口映射到宿主机：

```yaml {title="docker-compose.yml"}
services:
  app:
    image: your_image
    ports:
      - "0030:0030/udp"
```

改完后重启容器：

```bash
docker-compose down && docker-compose up -d
```

## 检查防火墙规则

很多时候流量是被防火墙规则拦截了，尤其是当容器需要和其他主机（而不是你本地电脑）通信时。

### 1. 检查 iptables 规则

运行下面的命令查看 iptables 规则：

```bash
sudo iptables -S
```

可能会看到这样的规则：

```bash
-A PREROUTING -d 168.17.5.20/32 ! -i localbridge -j DROP
-A DOCKER ! -i testbrige -o localbridge -j DROP
```

这些规则表示：发往容器 IP 的流量，除非来自 `localbridge` 网卡，否则都会被丢弃。这些规则是 Docker 每次启动时自动添加的，目前我还没找到绕过的办法。只能删掉它们，再用下面的规则覆盖（如果你在公共网络中，不建议这样做）：

```bash
sudo iptables -A DOCKER -j ACCEPT
```

> [!WARNING]
> 这条命令会允许所有流量进入 Docker 内部网络。如果你在公共网络中，可以在命令里指定允许 Docker 接收哪个网卡的流量。

### 2. 检查 nftables 规则

查看 nftables 规则：

```bash
nft list ruleset
```

输出示例：

```nft
table ip raw {
  chain PREROUTING {
    iifname != "localbridge" ip daddr 168.17.5.20 drop
    iifname != "localbridge" ip daddr 168.17.5.21 drop
    ...
  }
}
```

这些规则进一步限制了从非 `localbridge` 网卡访问容器 IP。它们会在运行 `docker-compose` 时自动生成，而且无法关闭。

> [!NOTE]
> 目前我只能每次都用脚本把它们删掉。

## 解决问题的关键步骤

### 删除 nftables 中的 DROP 规则

清空 `raw` 表中的 DROP 规则：

```bash
sudo nft flush chain ip raw PREROUTING
```

也可以根据你的环境只删除特定的规则。

### 确保容器服务监听在 `0.0.0.0:0030/udp`

容器内的服务必须监听在 `0.0.0.0` 或者具体网卡的 IP 上。

> [!CAUTION]
> 不要使用 `127.0.0.1`，因为它只允许本机访问。

## 工具安装

### netstat

安装 `net-tools` 包：

```bash
apt install net-tools
```

### nftables

安装 `nftables` 包：

```bash
apt install nftables
```

## 常用命令速查

- 查看 Docker 网络配置：
  ```bash
  docker network inspect <network-name>
  ```
- 查看监听的端口：
  ```bash
  ss -uln
  netstat -uln
  ```
- 清空特定的 nftables 规则：
  ```bash
  sudo nft flush chain ip raw PREROUTING
  ```
- 抓包确认通信情况：
  ```bash
  tcpdump -i <bridge_name> udp port 0030
  ```
- 查看路由的 MTU：
  ```bash
  ip route get <target-ip>
  ```

## 总结

使用自定义 Docker 网络通信时，常见的问题有：

- 服务没有监听在正确的网卡上。
- Docker bridge 网络没有开放需要的端口。
- iptables 或 nftables 规则默认丢弃了流量。
- VPN 隧道中的 MTU 限制。
- DNS 或 NAT 表配置错误。

按照上面的步骤操作，就可以解决这些问题，让容器与外部网络之间的 UDP 通信恢复正常。
