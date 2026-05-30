---
tags:
  - kvm
  - wireguard
  - routed-network
  - overlay-network
---

# KVM Multi-Host Network Lab: WireGuard Routed Subnets

## Overview

This solution gives every KVM host its own VM subnet and connects those subnets with WireGuard routes.

Use this when:

- VMs only need IP connectivity to each other.
- VMs do not need to be in the same L2 broadcast domain.
- You want a simple, secure, production-friendly design.

```text
Host A                          Host B                          Host C
VM subnet: 10.10.1.0/24         VM subnet: 10.10.2.0/24         VM subnet: 10.10.3.0/24
bridge: br-vm                   bridge: br-vm                   bridge: br-vm
wg0: 10.255.0.1/24              wg0: 10.255.0.2/24              wg0: 10.255.0.3/24
      \________________________ WireGuard mesh ________________________/
```

## Lab Assumptions

| Node | Host public/private underlay IP | WireGuard IP | VM subnet | Bridge IP |
|---|---:|---:|---:|---:|
| `kvm-host-a` | `192.168.100.11` | `10.255.0.1/24` | `10.10.1.0/24` | `10.10.1.1/24` |
| `kvm-host-b` | `192.168.100.12` | `10.255.0.2/24` | `10.10.2.0/24` | `10.10.2.1/24` |
| `kvm-host-c` | `192.168.100.13` | `10.255.0.3/24` | `10.10.3.0/24` | `10.10.3.1/24` |

Replace these IPs with your real host management IPs.

## 1. Install Packages On Every Host

Ubuntu/Debian:

```shell
sudo apt-get update
sudo apt-get install -y wireguard bridge-utils iproute2 dnsmasq
```

RHEL/CentOS/Rocky:

```shell
sudo dnf install -y wireguard-tools bridge-utils iproute dnsmasq
```

## 2. Create A VM Bridge On Every Host

Run on `kvm-host-a`:

```shell
sudo ip link add br-vm type bridge
sudo ip addr add 10.10.1.1/24 dev br-vm
sudo ip link set br-vm up
```

Run on `kvm-host-b`:

```shell
sudo ip link add br-vm type bridge
sudo ip addr add 10.10.2.1/24 dev br-vm
sudo ip link set br-vm up
```

Run on `kvm-host-c`:

```shell
sudo ip link add br-vm type bridge
sudo ip addr add 10.10.3.1/24 dev br-vm
sudo ip link set br-vm up
```

Persist this with Netplan, NetworkManager, or systemd-networkd for your Linux distribution after the lab works.

## 3. Enable Routing On Every Host

```shell
echo 'net.ipv4.ip_forward = 1' | sudo tee /etc/sysctl.d/99-kvm-routing.conf
sudo sysctl --system
```

Check:

```shell
sysctl net.ipv4.ip_forward
```

Expected:

```text
net.ipv4.ip_forward = 1
```

## 4. Generate WireGuard Keys

Run on every host:

```shell
sudo mkdir -p /etc/wireguard
umask 077
wg genkey | sudo tee /etc/wireguard/private.key | wg pubkey | sudo tee /etc/wireguard/public.key
sudo cat /etc/wireguard/public.key
```

Collect the public key from each host:

```text
HOST_A_PUBLIC_KEY=...
HOST_B_PUBLIC_KEY=...
HOST_C_PUBLIC_KEY=...
```

## 5. Configure WireGuard

On `kvm-host-a`, create `/etc/wireguard/wg0.conf`:

```ini
[Interface]
Address = 10.255.0.1/24
ListenPort = 51820
PrivateKey = HOST_A_PRIVATE_KEY

[Peer]
PublicKey = HOST_B_PUBLIC_KEY
Endpoint = 192.168.100.12:51820
AllowedIPs = 10.255.0.2/32, 10.10.2.0/24
PersistentKeepalive = 25

[Peer]
PublicKey = HOST_C_PUBLIC_KEY
Endpoint = 192.168.100.13:51820
AllowedIPs = 10.255.0.3/32, 10.10.3.0/24
PersistentKeepalive = 25
```

On `kvm-host-b`:

```ini
[Interface]
Address = 10.255.0.2/24
ListenPort = 51820
PrivateKey = HOST_B_PRIVATE_KEY

[Peer]
PublicKey = HOST_A_PUBLIC_KEY
Endpoint = 192.168.100.11:51820
AllowedIPs = 10.255.0.1/32, 10.10.1.0/24
PersistentKeepalive = 25

[Peer]
PublicKey = HOST_C_PUBLIC_KEY
Endpoint = 192.168.100.13:51820
AllowedIPs = 10.255.0.3/32, 10.10.3.0/24
PersistentKeepalive = 25
```

On `kvm-host-c`:

```ini
[Interface]
Address = 10.255.0.3/24
ListenPort = 51820
PrivateKey = HOST_C_PRIVATE_KEY

[Peer]
PublicKey = HOST_A_PUBLIC_KEY
Endpoint = 192.168.100.11:51820
AllowedIPs = 10.255.0.1/32, 10.10.1.0/24
PersistentKeepalive = 25

[Peer]
PublicKey = HOST_B_PUBLIC_KEY
Endpoint = 192.168.100.12:51820
AllowedIPs = 10.255.0.2/32, 10.10.2.0/24
PersistentKeepalive = 25
```

Replace `HOST_X_PRIVATE_KEY` with:

```shell
sudo cat /etc/wireguard/private.key
```

Protect the config:

```shell
sudo chmod 600 /etc/wireguard/wg0.conf
```

## 6. Allow WireGuard On Host Firewall

If using firewalld:

```shell
sudo firewall-cmd --add-port=51820/udp --permanent
sudo firewall-cmd --reload
```

If using ufw:

```shell
sudo ufw allow 51820/udp
```

## 7. Start WireGuard

Run on every host:

```shell
sudo systemctl enable --now wg-quick@wg0
sudo wg show
ip route
```

Expected routes on `kvm-host-a`:

```text
10.10.2.0/24 dev wg0
10.10.3.0/24 dev wg0
```

## 8. Attach VMs To The Local Bridge

Example libvirt interface XML:

```xml
<interface type='bridge'>
  <source bridge='br-vm'/>
  <model type='virtio'/>
</interface>
```

Attach to an existing VM:

```shell
virsh attach-interface VM_NAME \
  --type bridge \
  --source br-vm \
  --model virtio \
  --config \
  --live
```

Inside the VM on `kvm-host-a`, set an IP such as:

```shell
sudo ip addr add 10.10.1.10/24 dev ens3
sudo ip route replace default via 10.10.1.1
```

Inside a VM on `kvm-host-b`:

```shell
sudo ip addr add 10.10.2.10/24 dev ens3
sudo ip route replace default via 10.10.2.1
```

## 9. Test VM-To-VM Connectivity

From a VM on `kvm-host-a`:

```shell
ping -c 3 10.10.2.10
ping -c 3 10.10.3.10
curl -v http://10.10.2.10:PORT
```

From the KVM host:

```shell
sudo wg show
ip route get 10.10.2.10
```

## Troubleshooting

- Check host firewall allows UDP `51820`.
- Check each `AllowedIPs` includes the remote WireGuard IP and remote VM subnet.
- Check `net.ipv4.ip_forward=1`.
- Check VM default gateway points to the local bridge IP.
- Check no VM subnet overlaps with another VM subnet or the underlay host network.
- Use `tcpdump -ni wg0 icmp` to confirm packets enter the tunnel.

## Pros And Cons

Pros:

- Simple and secure.
- No L2 broadcast stretching.
- Easy to reason about routes.
- Good for production labs and small clusters.

Cons:

- VMs are not in the same subnet.
- Some software that requires L2 broadcast discovery may not work without extra configuration.

## Cleanup

```shell
sudo systemctl disable --now wg-quick@wg0
sudo rm -f /etc/wireguard/wg0.conf
sudo ip link delete br-vm
```

