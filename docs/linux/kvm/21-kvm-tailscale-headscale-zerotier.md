---
tags:
  - kvm
  - tailscale
  - headscale
  - zerotier
  - overlay-network
---

# KVM Multi-Host Network Lab: Tailscale, Headscale, Or ZeroTier

## Overview

This solution uses an existing overlay network product instead of building tunnels manually.

Good choices:

- **Tailscale**: easiest managed WireGuard-based overlay.
- **Headscale**: self-hosted control plane compatible with Tailscale clients.
- **ZeroTier**: mature SD-WAN style overlay with central network controller.

Use this when:

- You want the fastest lab.
- KVM hosts are behind NAT.
- You want simple admin access across sites.
- You do not need full L2 broadcast between VMs.

There are two common designs:

```text
Design A: install overlay client on every VM

VM A ----- overlay network ----- VM B ----- overlay network ----- VM C
```

```text
Design B: install overlay client on KVM hosts and advertise VM subnets

VM subnet A          VM subnet B          VM subnet C
10.40.1.0/24         10.40.2.0/24         10.40.3.0/24
     |                    |                    |
 KVM Host A -------- overlay mesh -------- KVM Host B/C
 subnet router       subnet router        subnet router
```

For KVM labs, Design B is usually cleaner because VMs do not need overlay agents.

## Lab Assumptions

| Node | VM subnet | Host bridge IP |
|---|---:|---:|
| `kvm-host-a` | `10.40.1.0/24` | `10.40.1.1/24` |
| `kvm-host-b` | `10.40.2.0/24` | `10.40.2.1/24` |
| `kvm-host-c` | `10.40.3.0/24` | `10.40.3.1/24` |

## 1. Prepare VM Bridges

On `kvm-host-a`:

```shell
sudo ip link add br-vm type bridge
sudo ip addr add 10.40.1.1/24 dev br-vm
sudo ip link set br-vm up
```

On `kvm-host-b`:

```shell
sudo ip link add br-vm type bridge
sudo ip addr add 10.40.2.1/24 dev br-vm
sudo ip link set br-vm up
```

On `kvm-host-c`:

```shell
sudo ip link add br-vm type bridge
sudo ip addr add 10.40.3.1/24 dev br-vm
sudo ip link set br-vm up
```

Enable routing on every host:

```shell
echo 'net.ipv4.ip_forward = 1' | sudo tee /etc/sysctl.d/99-kvm-overlay-routing.conf
sudo sysctl --system
```

Attach VMs to `br-vm`:

```shell
virsh attach-interface VM_NAME \
  --type bridge \
  --source br-vm \
  --model virtio \
  --config \
  --live
```

Set VM gateway to the local bridge IP.

## 2. Option A: Tailscale Subnet Router

Install Tailscale on every KVM host:

```shell
curl -fsSL https://tailscale.com/install.sh | sh
```

Advertise the local VM subnet.

On `kvm-host-a`:

```shell
sudo tailscale up --advertise-routes=10.40.1.0/24
```

On `kvm-host-b`:

```shell
sudo tailscale up --advertise-routes=10.40.2.0/24
```

On `kvm-host-c`:

```shell
sudo tailscale up --advertise-routes=10.40.3.0/24
```

In the Tailscale admin console:

1. Open **Machines**.
2. Select each KVM host.
3. Approve the advertised subnet route.

Test from a VM on host A to a VM on host B:

```shell
ping -c 3 10.40.2.10
```

Check Tailscale status:

```shell
tailscale status
tailscale netcheck
```

## 3. Option B: Headscale Subnet Router

Headscale is useful when you want a self-hosted control plane.

High-level lab layout:

```text
headscale-server: 192.168.100.10
kvm-host-a: Tailscale client joined to Headscale
kvm-host-b: Tailscale client joined to Headscale
kvm-host-c: Tailscale client joined to Headscale
```

Install and configure Headscale on one server following the official package instructions for your distribution.

After Headscale is running, create a user or namespace:

```shell
sudo headscale users create lab
```

On each KVM host, install the Tailscale client:

```shell
curl -fsSL https://tailscale.com/install.sh | sh
```

Join each KVM host to Headscale:

```shell
sudo tailscale up \
  --login-server=http://HEADSCALE_SERVER:8080 \
  --advertise-routes=10.40.1.0/24
```

Register the node on the Headscale server using the command printed by the client.

Approve or enable the subnet route in Headscale:

```shell
sudo headscale routes list
sudo headscale routes enable -r ROUTE_ID
```

Repeat with each host's own subnet.

## 4. Option C: ZeroTier Routed VM Subnets

Create a ZeroTier network in the ZeroTier controller.

Install ZeroTier on every KVM host:

```shell
curl -s https://install.zerotier.com | sudo bash
```

Join the network:

```shell
sudo zerotier-cli join ZEROTIER_NETWORK_ID
```

Approve the hosts in the ZeroTier web controller or your self-hosted controller.

In the ZeroTier network managed routes, add routes:

```text
10.40.1.0/24 via KVM_HOST_A_ZEROTIER_IP
10.40.2.0/24 via KVM_HOST_B_ZEROTIER_IP
10.40.3.0/24 via KVM_HOST_C_ZEROTIER_IP
```

On every KVM host, allow forwarding:

```shell
sudo sysctl -w net.ipv4.ip_forward=1
```

Check ZeroTier:

```shell
sudo zerotier-cli status
sudo zerotier-cli listnetworks
ip route
```

## 5. VM IP Configuration

Example VM on host A:

```shell
sudo ip addr add 10.40.1.10/24 dev ens3
sudo ip route replace default via 10.40.1.1
```

Example VM on host B:

```shell
sudo ip addr add 10.40.2.10/24 dev ens3
sudo ip route replace default via 10.40.2.1
```

Example VM on host C:

```shell
sudo ip addr add 10.40.3.10/24 dev ens3
sudo ip route replace default via 10.40.3.1
```

## 6. Test Connectivity

From VM on host A:

```shell
ping -c 3 10.40.2.10
ping -c 3 10.40.3.10
```

From host A:

```shell
ip route get 10.40.2.10
```

For Tailscale:

```shell
tailscale status
```

For ZeroTier:

```shell
sudo zerotier-cli peers
```

## Troubleshooting

- Confirm IP forwarding is enabled on every subnet-router host.
- Confirm the VM default gateway points to the local KVM bridge IP.
- Confirm subnet routes are approved in the overlay controller.
- Confirm VM subnet CIDRs do not overlap.
- Confirm host firewall allows forwarded traffic between the bridge and overlay interface.
- Check whether the overlay is using direct peer-to-peer paths or relay paths.

Example firewall check:

```shell
sudo iptables -S
sudo nft list ruleset
```

## Pros And Cons

Pros:

- Fastest to lab.
- Works well across NAT and home networks.
- Minimal manual tunnel configuration.
- Good for admin and cross-site private access.

Cons:

- Depends on an external or self-hosted control plane.
- Usually routed L3, not stretched L2.
- Product-specific ACLs and route approval need to be understood.
- Managed services may not fit strict compliance environments.

## Cleanup

Tailscale:

```shell
sudo tailscale down
sudo systemctl disable --now tailscaled
```

ZeroTier:

```shell
sudo zerotier-cli leave ZEROTIER_NETWORK_ID
sudo systemctl disable --now zerotier-one
```

Bridge cleanup:

```shell
sudo ip link delete br-vm
```

