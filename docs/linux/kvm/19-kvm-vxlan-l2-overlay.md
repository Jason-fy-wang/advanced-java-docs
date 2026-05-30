---
tags:
  - kvm
  - vxlan
  - l2-overlay
  - bridge
---

# KVM Multi-Host Network Lab: VXLAN L2 Overlay

## Overview

VXLAN lets VMs on different KVM hosts share the same Layer 2 network over an existing Layer 3 host network.

Use this when:

- VMs must be in the same subnet.
- You need broadcast, ARP, or L2 discovery between hosts.
- You want a lightweight overlay without a full SDN controller.

```text
VM A 10.20.0.11/24       VM B 10.20.0.12/24       VM C 10.20.0.13/24
        |                        |                        |
     br-vxlan                 br-vxlan                 br-vxlan
        |                        |                        |
    vxlan100                 vxlan100                 vxlan100
        \________________ same L2 segment ________________/
```

## Lab Assumptions

| Node | Host underlay IP | VXLAN bridge | VXLAN ID | VM subnet |
|---|---:|---|---:|---:|
| `kvm-host-a` | `192.168.100.11` | `br-vxlan` | `100` | `10.20.0.0/24` |
| `kvm-host-b` | `192.168.100.12` | `br-vxlan` | `100` | `10.20.0.0/24` |
| `kvm-host-c` | `192.168.100.13` | `br-vxlan` | `100` | `10.20.0.0/24` |

VXLAN uses UDP port `4789` by default.

## 1. Install Tools On Every Host

Ubuntu/Debian:

```shell
sudo apt-get update
sudo apt-get install -y bridge-utils iproute2
```

RHEL/CentOS/Rocky:

```shell
sudo dnf install -y bridge-utils iproute
```

## 2. Allow VXLAN UDP Port

If using firewalld:

```shell
sudo firewall-cmd --add-port=4789/udp --permanent
sudo firewall-cmd --reload
```

If using ufw:

```shell
sudo ufw allow 4789/udp
```

## 3. Create VXLAN On Host A

```shell
sudo ip link add br-vxlan type bridge
sudo ip link set br-vxlan up

sudo ip link add vxlan100 type vxlan \
  id 100 \
  local 192.168.100.11 \
  dstport 4789 \
  nolearning

sudo ip link set vxlan100 master br-vxlan
sudo ip link set vxlan100 up

sudo bridge fdb append 00:00:00:00:00:00 dev vxlan100 dst 192.168.100.12
sudo bridge fdb append 00:00:00:00:00:00 dev vxlan100 dst 192.168.100.13
```

## 4. Create VXLAN On Host B

```shell
sudo ip link add br-vxlan type bridge
sudo ip link set br-vxlan up

sudo ip link add vxlan100 type vxlan \
  id 100 \
  local 192.168.100.12 \
  dstport 4789 \
  nolearning

sudo ip link set vxlan100 master br-vxlan
sudo ip link set vxlan100 up

sudo bridge fdb append 00:00:00:00:00:00 dev vxlan100 dst 192.168.100.11
sudo bridge fdb append 00:00:00:00:00:00 dev vxlan100 dst 192.168.100.13
```

## 5. Create VXLAN On Host C

```shell
sudo ip link add br-vxlan type bridge
sudo ip link set br-vxlan up

sudo ip link add vxlan100 type vxlan \
  id 100 \
  local 192.168.100.13 \
  dstport 4789 \
  nolearning

sudo ip link set vxlan100 master br-vxlan
sudo ip link set vxlan100 up

sudo bridge fdb append 00:00:00:00:00:00 dev vxlan100 dst 192.168.100.11
sudo bridge fdb append 00:00:00:00:00:00 dev vxlan100 dst 192.168.100.12
```

## 6. Attach VMs To The VXLAN Bridge

Attach an existing VM:

```shell
virsh attach-interface VM_NAME \
  --type bridge \
  --source br-vxlan \
  --model virtio \
  --config \
  --live
```

Example VM IPs:

```text
VM on host A: 10.20.0.11/24
VM on host B: 10.20.0.12/24
VM on host C: 10.20.0.13/24
```

Inside each VM:

```shell
sudo ip addr add 10.20.0.11/24 dev ens3
sudo ip link set ens3 up
```

Use the correct IP for each VM.

## 7. Test L2 And L3 Connectivity

From VM on Host A:

```shell
ping -c 3 10.20.0.12
ping -c 3 10.20.0.13
arp -n
```

On the KVM hosts:

```shell
bridge link
bridge fdb show dev vxlan100
ip -d link show vxlan100
```

Use tcpdump:

```shell
sudo tcpdump -ni any udp port 4789
```

## 8. Optional: Give The Overlay A Gateway

If VMs need internet or upstream access, choose one host as the gateway.

On `kvm-host-a`:

```shell
sudo ip addr add 10.20.0.1/24 dev br-vxlan
sudo sysctl -w net.ipv4.ip_forward=1
```

Inside each VM:

```shell
sudo ip route replace default via 10.20.0.1
```

If outbound internet is required through host A:

```shell
sudo iptables -t nat -A POSTROUTING -s 10.20.0.0/24 -o UNDERLAY_NIC -j MASQUERADE
```

Replace `UNDERLAY_NIC` with `eth0`, `ens160`, or the actual host interface.

## Troubleshooting

- Confirm UDP `4789` is allowed between all KVM hosts.
- Confirm every host uses the same VXLAN ID.
- Confirm `local` is the real underlay IP of that host.
- Check MTU. VXLAN adds overhead; set VM MTU lower if packets drop.
- Avoid loops. Do not bridge the same L2 segment through multiple paths without STP planning.
- If ARP fails, check `bridge fdb show dev vxlan100`.

## Pros And Cons

Pros:

- VMs can share one subnet across hosts.
- No controller required for a small lab.
- Works with standard Linux bridge.

Cons:

- Broadcast traffic is stretched across hosts.
- Static FDB entries become painful as host count grows.
- MTU problems are common if the underlay does not support larger frames.
- L2 failures are harder to troubleshoot than routed networks.

## Cleanup

Run on every host:

```shell
sudo ip link delete vxlan100
sudo ip link delete br-vxlan
```

