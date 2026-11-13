---
title: "Docker Container VLAN Forwarding"
date: "2025-11-21"
tags: ["Network", "VLAN", "Docker"]
---

In some industrial or legacy network environments, certain devices (like QNX systems) require traffic to be tagged with specific VLANs. Docker containers, by default, do not send VLAN-tagged traffic directly. This article explains the principle behind and introduces how to solve this problem.

## What is VLAN

VLAN (Virtual Local Area Network) is a Layer 2 ([OSI Layer](https://en.wikipedia.org/wiki/OSI_model#Layer_2:_Data_link_layer)) technology that segments a physical network into multiple logical networks. Each VLAN has a unique ID (802.1Q tag). Devices in different VLANs cannot communicate directly unless routed.

### Why docker not support VLAN

1. Docker’s default networking model is Layer-3 (IP-based), not Layer-2

VLAN (IEEE 802.1Q) is strictly a Layer 2 technology. But Docker’s default network drivers (bridge, overlay, host, macvlan, ipvlan):

- manage IP routing
- assign IP addresses
- perform NAT, routing, forwarding
- abstract away L2 frames

Docker hides or virtualizes L2 details so containers don’t manage Ethernet frames directly.

2. Docker "bridge" networks use a virtual Ethernet switch that strips VLAN tags

The Docker bridge (docker0) is implemented using Linux bridge: Linux bridges drop VLAN tags unless VLAN filtering is explicitly enabled
Docker configures the bridge in a simple L3 routing mode. Docker does not configure VLAN awareness or per-port VLAN mapping. So any VLAN-tagged frame from a container would be lost at the bridge.

3. VLAN requires direct control of Ethernet frames, but Docker’s drivers filter or encapsulate them

- bridge driver: Strips VLAN tags at the Linux bridge.
- overlay driver: Encapsulates traffic in VXLAN, discarding any L2 metadata like VLAN tags.
- macvlan / ipvlan: Passes L2 but only the host can add VLAN subinterfaces, not Docker.

So Docker never provides a path where containers can generate raw 802.1Q frames. Even when macvlan is used, VLANs must be configured on the host, not inside Docker.

### Why GRE tunnel(AWS) not support VLAN

AWS’s GRE implementation (used in TGW Connect, Direct Connect + GRE, SD-WAN integrations, etc.) operates strictly as a Layer-3 (L3) GRE encapsulation service. Because of this architectural choice, VLAN tags (802.1Q headers) are intentionally stripped or rejected.

AWS GRE tunnels only carry:

- IPv4 packets
- IPv6 packets (depending on service)

GRE frames inside AWS do not support Layer-2 payloads, such as:

- Ethernet frames
- MAC addresses
- 802.1Q VLAN headers
- ARP, LLDP, STP, etc.

Since a VLAN tag is part of the Ethernet header, and AWS’s GRE transport never transports Ethernet frames, the VLAN tag has no place to exist.

## How to solve

To enable containers to communicate with VLAN-aware devices, we need to

1. Create VLAN subinterface on host:

```bash
sudo ip link add link eth0 name eth0.100 type vlan id 100
sudo ip link set dev eth0.100 up
```

> [!NOTE]
> Ensure firewall and network policies allow rewritten IP/MAC traffic.

2. Add a bridge between the original untagged interface(eth0) with the tagged interface(eth0.100)

```bash
sudo ip link add name br0 type bridge
sudo ip link set br0 up
sudo ip link set eth0 master br0
sudo ip link set eth0.100 master br0
```

Now traffic arriving on eth0 (untagged) will be bridged to eth0.100, which adds VLAN 100 tag.
Traffic arriving on eth0.100 (tagged with VLAN 100) will be bridged to eth0 and tag will be removed.

## Advanced suggestions

If you want to redirect the network to a outside devices, you could use Linux Traffic Control to rewrite the whole L2/L3 headers (MAC, IP, TTL)
and that would give the possibilities to redirect traffic to the physical interface.

For example:

```bash
tc filter add dev eth0.100 ingress protocol ip pref 10 flower \
  ip_proto udp src_ip "$SRC_IP" dst_ip "$DST_IP" \
  src_port "$SRC_PORT" dst_port "$DST_PORT" \
  action pedit ex munge ip src set "$SRC_IP_NEW" pipe \
  ...
```
