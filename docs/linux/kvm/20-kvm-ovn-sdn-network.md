---
tags:
  - kvm
  - ovn
  - openvswitch
  - sdn
---

# KVM Multi-Host Network Lab: Open vSwitch And OVN

## Overview

OVN is a software-defined networking system built on Open vSwitch. It gives KVM hosts cloud-style virtual networking:

- Logical switches
- Logical routers
- Distributed routing
- DHCP options
- ACLs
- NAT

Use this when:

- You want a small private-cloud network lab.
- You need multiple tenant networks.
- You want SDN-style security policies and logical routing.
- You are comfortable with more moving parts.

```text
OVN Central
  northbound DB
  southbound DB
  ovn-northd
      |
      | control plane
      v
KVM Host A              KVM Host B              KVM Host C
OVS + ovn-controller    OVS + ovn-controller    OVS + ovn-controller
VM ports                VM ports                VM ports
```

## Lab Assumptions

| Node | Role | Underlay IP |
|---|---|---:|
| `ovn-central` | OVN central databases and `ovn-northd` | `192.168.100.10` |
| `kvm-host-a` | KVM + OVS + OVN controller | `192.168.100.11` |
| `kvm-host-b` | KVM + OVS + OVN controller | `192.168.100.12` |
| `kvm-host-c` | KVM + OVS + OVN controller | `192.168.100.13` |

For a small lab, `ovn-central` can run on `kvm-host-a`. For a cleaner design, keep it separate.

## 1. Install Packages

Package names vary by distribution.

Ubuntu/Debian:

```shell
sudo apt-get update
sudo apt-get install -y openvswitch-switch ovn-central ovn-host ovn-common libvirt-daemon-system
```

RHEL/CentOS/Rocky:

```shell
sudo dnf install -y openvswitch ovn-central ovn-host ovn libvirt
```

Enable OVS on every KVM host:

```shell
sudo systemctl enable --now openvswitch-switch || sudo systemctl enable --now openvswitch
```

## 2. Start OVN Central

Run on `ovn-central`:

```shell
sudo systemctl enable --now ovn-central
```

Check listening ports:

```shell
sudo ss -lntup | grep 664
```

Expected control-plane ports:

```text
6641 northbound DB
6642 southbound DB
```

If your package does not expose remote DB listeners by default, configure OVN central to listen on TCP for the lab network.

Example:

```shell
sudo ovn-nbctl set-connection ptcp:6641:0.0.0.0
sudo ovn-sbctl set-connection ptcp:6642:0.0.0.0
```

Restrict these ports with host firewall rules in production.

## 3. Configure Each KVM Host As An OVN Chassis

Run on `kvm-host-a`:

```shell
sudo ovs-vsctl set open . external_ids:ovn-remote=tcp:192.168.100.10:6642
sudo ovs-vsctl set open . external_ids:ovn-encap-type=geneve
sudo ovs-vsctl set open . external_ids:ovn-encap-ip=192.168.100.11
sudo systemctl enable --now ovn-host
```

Run on `kvm-host-b`:

```shell
sudo ovs-vsctl set open . external_ids:ovn-remote=tcp:192.168.100.10:6642
sudo ovs-vsctl set open . external_ids:ovn-encap-type=geneve
sudo ovs-vsctl set open . external_ids:ovn-encap-ip=192.168.100.12
sudo systemctl enable --now ovn-host
```

Run on `kvm-host-c`:

```shell
sudo ovs-vsctl set open . external_ids:ovn-remote=tcp:192.168.100.10:6642
sudo ovs-vsctl set open . external_ids:ovn-encap-type=geneve
sudo ovs-vsctl set open . external_ids:ovn-encap-ip=192.168.100.13
sudo systemctl enable --now ovn-host
```

Allow Geneve UDP `6081` between KVM hosts:

```shell
sudo firewall-cmd --add-port=6081/udp --permanent
sudo firewall-cmd --reload
```

## 4. Verify OVN Chassis Registration

Run on `ovn-central`:

```shell
sudo ovn-sbctl show
```

Expected:

```text
Chassis kvm-host-a
Chassis kvm-host-b
Chassis kvm-host-c
```

## 5. Create An OVN Logical Switch

Run on `ovn-central`:

```shell
sudo ovn-nbctl ls-add ls-kvm

sudo ovn-nbctl set logical_switch ls-kvm \
  other_config:subnet=10.30.0.0/24
```

Create VM logical ports:

```shell
sudo ovn-nbctl lsp-add ls-kvm vm-a-port
sudo ovn-nbctl lsp-set-addresses vm-a-port "02:00:00:00:00:11 10.30.0.11"

sudo ovn-nbctl lsp-add ls-kvm vm-b-port
sudo ovn-nbctl lsp-set-addresses vm-b-port "02:00:00:00:00:12 10.30.0.12"

sudo ovn-nbctl lsp-add ls-kvm vm-c-port
sudo ovn-nbctl lsp-set-addresses vm-c-port "02:00:00:00:00:13 10.30.0.13"
```

## 6. Prepare The OVS Integration Bridge

On each KVM host, create an OVS integration bridge if it does not exist. `ovn-controller` programs this bridge.

```shell
sudo ovs-vsctl --may-exist add-br br-int
```

Do not create `type=internal` ports for libvirt VMs. Libvirt creates the VM tap interfaces, usually named `vnet0`, `vnet1`, and so on. OVN will only bind the logical switch port when the real VM tap interface on `br-int` has `external_ids:iface-id` set to the matching OVN logical switch port name.

## 7. Attach VM Interfaces

Attach VM tap devices to OVS through libvirt using an Open vSwitch bridge.

Example VM XML interface:

```xml
<interface type='bridge'>
  <mac address='02:00:00:00:00:11'/>
  <source bridge='br-int'/>
  <target dev='vnet-a'/>
  <virtualport type='openvswitch'/>
  <model type='virtio'/>
</interface>
```

Do not set `<parameters interfaceid='vm-a-port'/>` in this XML. Libvirt expects `interfaceid` to be a UUID, while this lab uses human-readable OVN logical port names such as `vm-a-port`.

the original config as this:
```xml
    <interface type='bridge'>
      <mac address='52:54:00:e3:29:f6'/>
      <source bridge='virbr0'/>
      <model type='virtio'/>
      <address type='pci' domain='0x0000' bus='0x01' slot='0x00' function='0x0'/>
    </interface>
```

Edit a VM:

```shell
virsh edit VM_NAME
```

Set the VM MAC to match the OVN logical switch port address.

After the VM starts, bind the libvirt tap interface to the OVN logical switch port.

On `kvm-host-a` for `vm-a`:

```shell
sudo ovs-vsctl set Interface vnet-a external_ids:iface-id=vm-a-port
sudo ovs-vsctl set Interface vnet-a external_ids:attached-mac=02:00:00:00:00:11
sudo ovs-vsctl set Interface vnet-a external_ids:vm-id=vm-a
sudo ovs-vsctl set Interface vnet-a external_ids:iface-status=active
```

On `kvm-host-b` for `vm-b`:

```shell
sudo ovs-vsctl set Interface vnet-b external_ids:iface-id=vm-b-port
sudo ovs-vsctl set Interface vnet-b external_ids:attached-mac=02:00:00:00:00:12
sudo ovs-vsctl set Interface vnet-b external_ids:vm-id=vm-b
```

On `kvm-host-c` for `vm-c`:

```shell
sudo ovs-vsctl set Interface vnet-c external_ids:iface-id=vm-c-port
sudo ovs-vsctl set Interface vnet-c external_ids:attached-mac=02:00:00:00:00:13
sudo ovs-vsctl set Interface vnet-c external_ids:vm-id=vm-c
```

If libvirt creates a different tap name, find it with:

```shell
sudo ovs-vsctl list-ports br-int
```

Inside `vm-a`:

```shell
sudo ip addr add 10.30.0.11/24 dev ens3
sudo ip link set ens3 up
```

Inside `vm-b`:

```shell
sudo ip addr add 10.30.0.12/24 dev ens3
sudo ip link set ens3 up
```

Inside `vm-c`:

```shell
sudo ip addr add 10.30.0.13/24 dev ens3
sudo ip link set ens3 up
```

## 8. Test VM Connectivity

From `vm-a`:

```shell
ping -c 3 10.30.0.12
ping -c 3 10.30.0.13
```

On KVM hosts:

```shell
sudo ovs-vsctl show
sudo ovs-ofctl show br-int
```

On `ovn-central`:

```shell
sudo ovn-nbctl show
sudo ovn-sbctl show
```

Trace a packet through OVN:

```shell
sudo ovn-trace ls-kvm 'inport == "vm-a-port" && eth.src == 02:00:00:00:00:11 && eth.dst == 02:00:00:00:00:12 && ip4.src == 10.30.0.11 && ip4.dst == 10.30.0.12 && ip.ttl == 64'
```

At this point the lab only provides tenant-to-tenant connectivity on `10.30.0.0/24`. It does not provide Internet egress yet. If `vm-a`, `vm-b`, or `vm-c` cannot reach `baidu.com`, that is expected until you configure either:

- the optional OVN logical router in section 9, or
- the host NAT gateway in section 10.

To reach public sites, each VM also needs:

- a default route via `10.30.0.1`
- working DNS, for example `/etc/resolv.conf` with a nameserver entry

## 9. Optional: Add A Logical Router

Create a logical router and gateway address:

```shell
sudo ovn-nbctl lr-add lr-kvm

sudo ovn-nbctl lrp-add lr-kvm lr-kvm-ls-kvm 02:00:00:00:ff:01 10.30.0.1/24

sudo ovn-nbctl lsp-add ls-kvm ls-kvm-lr-kvm
sudo ovn-nbctl lsp-set-type ls-kvm-lr-kvm router
sudo ovn-nbctl lsp-set-addresses ls-kvm-lr-kvm router
sudo ovn-nbctl lsp-set-options ls-kvm-lr-kvm router-port=lr-kvm-ls-kvm
```

Inside each VM:

```shell
sudo ip route replace default via 10.30.0.1
```

## 10. Optional: Give VMs Internet Access Through A KVM Host

Use this simple NAT approach when the KVM host already has internet access and you want VMs on `ls-kvm` to use that host as their default gateway.

Do not use this host gateway with the optional OVN logical router on the same gateway IP. Both examples use `10.30.0.1`; choose either the Linux host NAT gateway in this section or the OVN logical router in the previous section.

Example:

- VM network: `10.30.0.0/24`
- Host gateway IP on the OVN network: `10.30.0.1`
- Host internet interface: `eth0`

Replace `eth0` with the real internet-facing interface on your KVM host:

```shell
ip route get 8.8.8.8
```

### 10.1 Create A Host Gateway Port In OVN

Run on `ovn-central`:

```shell
sudo ovn-nbctl lsp-add ls-kvm host-gw-port
sudo ovn-nbctl lsp-set-addresses host-gw-port "02:00:00:00:ff:01 10.30.0.1"
```

Run on the KVM host that will provide internet access:

```shell
sudo ovs-vsctl --may-exist add-port br-int host-gw \
  -- set Interface host-gw type=internal \
  -- set Interface host-gw external_ids:iface-id=host-gw-port

sudo ip link set host-gw address 02:00:00:00:ff:01
sudo ip addr add 10.30.0.1/24 dev host-gw
sudo ip link set host-gw up
```

Validate that the host can reach the VMs:

```shell
ping -c 3 10.30.0.11
ping -c 3 10.30.0.12
```

### 10.2 Enable Linux Forwarding On The Host

Run on the NAT host:

```shell
sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward = 1" | sudo tee /etc/sysctl.d/99-ovn-vm-forwarding.conf
sudo sysctl --system
```

### 10.3 Add NAT On The Host

Using `iptables`:

```shell
INTERNET_IFACE="eth0"

sudo iptables -t nat -A POSTROUTING -s 10.30.0.0/24 -o "${INTERNET_IFACE}" -j MASQUERADE
sudo iptables -A FORWARD -i host-gw -o "${INTERNET_IFACE}" -s 10.30.0.0/24 -j ACCEPT
sudo iptables -A FORWARD -i "${INTERNET_IFACE}" -o host-gw -d 10.30.0.0/24 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
```

Using `firewalld`:

```shell
sudo firewall-cmd --add-masquerade --permanent
sudo firewall-cmd --reload
```

If `firewalld` is zone-based, put `host-gw` in a trusted or internal zone and keep the internet interface in the external zone:

```shell
sudo firewall-cmd --zone=trusted --add-interface=host-gw --permanent
sudo firewall-cmd --zone=external --add-masquerade --permanent
sudo firewall-cmd --reload
```

### 10.4 Configure The VM Default Route And DNS

Inside each VM:

```shell
sudo ip route replace default via 10.30.0.1
```

If DNS is not already configured:

```shell
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
```

Validate from a VM:

```shell
ping -c 3 10.30.0.1
ping -c 3 8.8.8.8
curl -I https://www.google.com
```

### 10.5 Troubleshoot VM Internet Access

If the host can ping VMs but VMs cannot reach the internet, check these in order:

1. VM default route:

```shell
ip route
```

Expected:

```text
default via 10.30.0.1 dev ens3
```

2. Host forwarding:

```shell
sysctl net.ipv4.ip_forward
```

Expected:

```text
net.ipv4.ip_forward = 1
```

3. Host NAT rule:

```shell
sudo iptables -t nat -S POSTROUTING
```

Expected rule includes:

```text
-s 10.30.0.0/24 -o eth0 -j MASQUERADE
```

4. OVN host gateway binding:

```shell
sudo ovn-sbctl list Port_Binding host-gw-port
sudo ovs-vsctl list Interface host-gw
```

The `Port_Binding` `chassis` field should not be empty, and the OVS interface should include:

```text
external_ids        : {iface-id="host-gw-port"}
```

5. Return path and firewall:

```shell
sudo tcpdump -ni host-gw icmp
sudo tcpdump -ni eth0 icmp
```

If packets appear on `host-gw` but not on `eth0`, the host forwarding or firewall rules are blocking traffic. If packets leave `eth0` but do not return, check upstream routing, host firewall, or cloud/security-group rules.

## Troubleshooting

### `Destination Host Unreachable` Between VMs

Example error:

```text
PING 10.30.0.11 (10.30.0.11) 56(84) bytes of data.
From 10.30.0.12 icmp_seq=1 Destination Host Unreachable
```

This usually means the VM cannot resolve the destination MAC address on the OVN logical switch. In this lab, the most common causes are:

- The real libvirt tap interface is not bound to the OVN logical switch port.
- The VM MAC or IP does not match `ovn-nbctl lsp-set-addresses`.
- Geneve UDP `6081` is blocked between KVM hosts.
- `ovn-controller` did not bind the logical port to a chassis.

Check the VM interface inside the source VM:

```shell
ip addr show ens3
ip neigh show
```

On the source KVM host, confirm the VM tap is attached to `br-int`:

```shell
sudo ovs-vsctl list-ports br-int
sudo ovs-vsctl list Interface vnet-b
```

The interface must include the matching `iface-id`. For `vm-b`, it should look like:

```text
external_ids        : {attached-mac="02:00:00:00:00:12", iface-id="vm-b-port", vm-id="vm-b"}
```

If `iface-id` is missing, set it on the real tap interface:

```shell
sudo ovs-vsctl set Interface vnet-b external_ids:iface-id=vm-b-port
sudo ovs-vsctl set Interface vnet-b external_ids:attached-mac=02:00:00:00:00:12
sudo ovs-vsctl set Interface vnet-b external_ids:vm-id=vm-b
```

On `ovn-central`, check that OVN bound the logical ports:

```shell
sudo ovn-sbctl list Port_Binding vm-a-port
sudo ovn-sbctl list Port_Binding vm-b-port
```

The `chassis` field should not be empty. If it is empty, check `ovn-controller` on the KVM host:

```shell
sudo systemctl status ovn-host
sudo ovs-vsctl get open . external_ids
```

Check that the OVN logical port addresses match the VM configuration:

```shell
sudo ovn-nbctl lsp-get-addresses vm-a-port
sudo ovn-nbctl lsp-get-addresses vm-b-port
```

For this lab, `vm-a` should be `02:00:00:00:00:11 10.30.0.11` and `vm-b` should be `02:00:00:00:00:12 10.30.0.12`.

- Check OVN chassis exist in `ovn-sbctl show`.
- Check each VM port has a matching `external_ids:iface-id`.
- Check VM MAC address matches `lsp-set-addresses`.
- Check Geneve UDP `6081` is allowed between hosts.
- Check `ovn-controller` is running on every KVM host.
- Check `br-int` exists and has the VM tap/interface attached.

## Pros And Cons

Pros:

- Most powerful option.
- Supports multi-tenant logical networks.
- Supports ACLs, routers, NAT, and DHCP-like options.
- Good foundation for OpenStack-style networking.

Cons:

- More complex control plane.
- More services to monitor.
- Requires stronger operational knowledge than WireGuard or simple VXLAN.

## Cleanup

On `ovn-central`:

```shell
sudo ovn-nbctl ls-del ls-kvm
sudo ovn-nbctl lr-del lr-kvm
```

On each KVM host:

```shell
sudo systemctl disable --now ovn-host
sudo ovs-vsctl del-br br-int
```
