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

## 6. Create OVS Ports For VMs

On each KVM host, create an OVS integration bridge if it does not exist:

```shell
sudo ovs-vsctl --may-exist add-br br-int
```

On `kvm-host-a`:

```shell
sudo ovs-vsctl add-port br-int vm-a-port \
  -- set Interface vm-a-port type=internal \
  -- set Interface vm-a-port external_ids:iface-id=vm-a-port
```

On `kvm-host-b`:

```shell
sudo ovs-vsctl add-port br-int vm-b-port \
  -- set Interface vm-b-port type=internal \
  -- set Interface vm-b-port external_ids:iface-id=vm-b-port
```

On `kvm-host-c`:

```shell
sudo ovs-vsctl add-port br-int vm-c-port \
  -- set Interface vm-c-port type=internal \
  -- set Interface vm-c-port external_ids:iface-id=vm-c-port
```

## 7. Attach VM Interfaces

For a simple lab, attach VM tap devices to OVS through libvirt using an Open vSwitch bridge.

Example VM XML interface:

```xml
<interface type='bridge'>
  <mac address='02:00:00:00:00:11'/>
  <source bridge='br-int'/>
  <virtualport type='openvswitch'>
    <parameters interfaceid='vm-a-port'/>
  </virtualport>
  <model type='virtio'/>
</interface>
```

Edit a VM:

```shell
virsh edit VM_NAME
```

Set the VM MAC to match the OVN logical switch port address.

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

## Troubleshooting

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

