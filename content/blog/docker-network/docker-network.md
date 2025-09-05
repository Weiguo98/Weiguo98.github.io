---
title: "Solution for External Network Unable to Communicate with Docker Containers"
date: "2025-09-05"
tags: ["Docker", "Network", "Troubleshoot"]
---

This technical document provides a step-by-step guide to troubleshoot and resolve issues where an external network cannot communicate with Docker containers, particularly for UDP traffic.

## Background

When using `docker-compose` to start a group of containers with a custom bridge network (e.g., `test-network`), you might encounter issues where UDP packets are not transmitted correctly between containers or between containers and external services. By analyzing `iptables`, `nftables`, and container listening states, you can configure the containers to properly listen and respond to UDP packets on specific ports.

## Network Structure Overview

To inspect your Docker network, use the following command:

```bash
docker network inspect test-network
```

The output might include key configurations such as:

```json
{
    "Subnet": "168.17.5.0/22",
    "Gateway": "168.17.7.253",
    "Options": {
        # the bridge network name you would like to connect in your local.
        "com.docker.network.bridge.name": "localbridge"
    }
}
```

Example container IPs in this network:

- 168.17.7.254
- 168.17.5.1
- 168.17.5.20

### Problem Description

Common issues observed in this scenario include:

- UDP packets sent from one container to an external IP (e.g., `168.19.1.1`) receive no response.
- Pinging container IPs does not work.
- Using `tcpdump` shows that packets are sent but not replied to.
- Packet capture logs indicate errors such as: udp port 0030 unreachable

## Troubleshooting Steps

### 1. Check Container Listening Ports

Inside the container, run the following command to check which ports are being listened to:

```bash
netstat -uln
```

If the output shows something like:

```
127.0.0.11:34321
```

This indicates that the container service is not listening on the actual business port (e.g., `0030`) or on all network interfaces.

### 2. Modify Container Service to Listen on the Correct Ports

Ensure that the service inside the container is configured to listen on the appropriate address and port, such as:

- `0.0.0.0:0030` (all interfaces)
- Or a specific IP, e.g., `168.17.5.x:0030`

You can verify this using:

```bash
ss -uln
```

### 3. Open UDP Ports in `docker-compose`

#### For Internal Container Communication

Use the `expose` directive in your `docker-compose.yml` file:

```yaml
services:
  app:
    image: your_image
    expose:
      - "0030/udp"
```

#### For Host-to-Container Communication

Use the `ports` directive to map the container port to the host:

```yaml
services:
  app:
    image: your_image
    ports:
      - "0030:0030/udp"
```

After making these changes, restart the containers:

```bash
docker-compose down && docker-compose up -d
```

## Firewall Rules Inspection

Many times, the traffic is banned by the firewalls rules, especially if you are connecting containers with other hosts(not in your local laptop).

### 1. Check iptables Rules

Run the following command to inspect iptables rules:

```bash
sudo iptables -S
```

You might find rules like:

```bash
-A PREROUTING -d 168.17.5.20/32 ! -i localbridge -j DROP
-A DOCKER ! -i testbrige -o localbridge -j DROP
```

These rules indicate that traffic targeting container IPs is dropped unless it originates from the `localbridge` interface. These rules are added automatically by docker whenever docker started. So far I haven't found anyway to go around with it. You have delete them and override them by (not recommend if you are in a public network):

```bash
sudo iptables -A DOCKER -j ACCEPT
```

This command will let any traffic go into docker internal network. If you are in a public network, you could specify which interface you would like docker to accept in the command.

### 2. Check nftables Rules

Inspect nftables rules with:

```bash
nft list ruleset
```

Example output:

```nft
table ip raw {
  chain PREROUTING {
    iifname != "localbridge" ip daddr 168.17.5.20 drop
    iifname != "localbridge" ip daddr 168.17.5.21 drop
    ...
  }
}
```

These rules further restrict access to container IPs from non-`localbridge` interfaces. These are automatically generated when you run `docker-compose` and there is no way to disable it. Currently, I have to use script to delete them every time.

## Key Steps to Resolve the Issue

### Remove DROP Rules in nftables

To clear the DROP rules in the `raw` table, use:

```bash
sudo nft flush chain ip raw PREROUTING
```

Alternatively, delete specific rules based on your setup.

### Ensure Container Services Listen on `0.0.0.0:0030/udp`

The service inside the container must listen on `0.0.0.0` or the specific network interface IP. Avoid using `127.0.0.1` as it restricts access to localhost.

## Tools and Installation

### netstat

Install the `net-tools` package:

```bash
apt install net-tools
```

### nftables

Install the `nftables` package:

```bash
apt install nftables
```

## Common Commands Reference

- Inspect Docker network configuration:
  ```bash
  docker network inspect <network-name>
  ```
- Check listening ports:
  ```bash
  ss -uln
  netstat -uln
  ```
- Clear specific nftables rules:
  ```bash
  sudo nft flush chain ip raw PREROUTING
  ```
- Capture packets to verify communication:
  ```bash
  tcpdump -i <bridge_name> udp port 0030
  ```
- Check route MTU:
  ```bash
  ip route get <target-ip>
  ```

## Summary

When setting up custom Docker networks for communications, common issues include:

- Services not listening on the correct network interfaces.
- Docker bridge networks not exposing required ports.
- iptables or nftables rules dropping traffic by default.
- MTU limitations in VPN tunnels.
- DNS or NAT table misconfigurations.

By following the steps outlined above, you can resolve these issues and enable seamless UDP communication between containers and external networks.
