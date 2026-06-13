---
tags:
  - kvm
  - ovs
  - openvswitch
  - ha
---

# Open vSwitch HA For KVM

## Overview

Open vSwitch does not make several KVM hosts into one HA switch by itself. In a
KVM environment, OVS HA usually means:

- Redundant physical NICs on each KVM host
- OVS bonding or Linux bonding under an OVS bridge
- Redundant upstream switches
- Optional LACP for active-active uplinks
- Stable bridge configuration for libvirt and OVN
- Clear failover tests

This document focuses on OVS host-side HA for KVM networking.

```text
                 upstream switch A      upstream switch B
                       |                       |
                       |                       |
                    eno1                    eno2
                       \                     /
                        \                   /
                         +---- bond0 ------+
                               |
                            br-provider
                               |
                        VM / OVN / VLAN traffic
```

## HA Design Choices

| Mode | Use case | Notes |
|---|---|---|
| `active-backup` | Simple HA across two switches | One link forwards, one standby |
| `balance-slb` | Load sharing without switch LACP | OVS balances by source MAC/VLAN |
| `balance-tcp` | LACP active-active | Requires upstream switch port-channel |

Recommended starting point:

- Use `active-backup` for simple and predictable failover.
- Use `balance-tcp` only when the physical switches are configured for LACP.
- Do not use `balance-tcp` without switch-side LACP.

## Lab Assumptions

| Item | Value |
|---|---|
| KVM host | `kvm-host-a` |
| Uplink NIC 1 | `eno1` |
| Uplink NIC 2 | `eno2` |
| Provider bridge | `br-provider` |
| OVS bond port | `bond-provider` |
| Native management VLAN | untagged or host-specific |
| VM VLAN example | `1927` |

Repeat the same pattern on every KVM host.

Important: changing uplink and bridge configuration can disconnect the host.
Use console access, IPMI, iDRAC, iLO, or out-of-band access before changing a
remote production host.

## 1. Install And Enable OVS

Ubuntu/Debian:

```shell
sudo apt-get update
sudo apt-get install -y openvswitch-switch
sudo systemctl enable --now openvswitch-switch
```

RHEL/CentOS/Rocky:

```shell
sudo dnf install -y openvswitch
sudo systemctl enable --now openvswitch
```

Verify:

```shell
sudo ovs-vsctl show
sudo systemctl status openvswitch-switch || sudo systemctl status openvswitch
```

## 2. Create An OVS Bridge

Create the provider bridge:

```shell
sudo ovs-vsctl --may-exist add-br br-provider
sudo ip link set br-provider up
```

Check:

```shell
sudo ovs-vsctl show
ip -br link show br-provider
```

## 3. Create An Active-Backup OVS Bond

Use this mode when the two NICs connect to separate switches without MLAG or
LACP.

```shell
sudo ovs-vsctl --may-exist add-bond br-provider bond-provider eno1 eno2 \
  bond_mode=active-backup \
  other_config:bond-primary=eno1
```

Bring the member links up:

```shell
sudo ip link set eno1 up
sudo ip link set eno2 up
```

Verify:

```shell
sudo ovs-appctl bond/show bond-provider
sudo ovs-vsctl list port bond-provider
```

Expected:

```text
bond_mode: active-backup
active slave mac: ...
```

## 4. Optional: Create An LACP Bond

Use this only when the upstream switches are configured for LACP. The switch
ports must be in the same port-channel. If the two switches are separate
devices, they must support MLAG, vPC, stacking, IRF, or an equivalent feature.

```shell
sudo ovs-vsctl --may-exist add-bond br-provider bond-provider eno1 eno2 \
  bond_mode=balance-tcp \
  lacp=active \
  other_config:lacp-time=fast
```

Verify LACP:

```shell
sudo ovs-appctl lacp/show bond-provider
sudo ovs-appctl bond/show bond-provider
```

Healthy output should show both members attached and synchronized.

If LACP is not healthy, traffic can blackhole. Fix the switch configuration or
use `active-backup`.

## 5. Optional: Use VLAN Trunks

Allow all VLANs on the provider bond:

```shell
sudo ovs-vsctl set port bond-provider vlan_mode=trunk
```

Restrict trunk VLANs:

```shell
sudo ovs-vsctl set port bond-provider trunks=100,200,1927
```

Show port configuration:

```shell
sudo ovs-vsctl list port bond-provider
```

Attach a VM tap port to VLAN `1927`:

```shell
sudo ovs-vsctl set port vnet3 tag=1927
```

Check:

```shell
sudo ovs-vsctl get port vnet3 tag
```

## 6. Move Host Management IP To The OVS Bridge

If the host management IP currently lives on `eno1`, move it to `br-provider`
or to a VLAN interface above `br-provider`.

Temporary runtime example for an untagged management network:

```shell
sudo ip addr flush dev eno1
sudo ip addr add 192.168.100.11/24 dev br-provider
sudo ip route replace default via 192.168.100.1
```

For a tagged management VLAN, create an internal OVS port:

```shell
sudo ovs-vsctl --may-exist add-port br-provider mgmt100 tag=100 \
  -- set interface mgmt100 type=internal
sudo ip link set mgmt100 up
sudo ip addr add 192.168.100.11/24 dev mgmt100
sudo ip route replace default via 192.168.100.1
```

Important: these `ip` commands are temporary. Make the final configuration
persistent with the host network manager.

## 7. Persist Configuration

The exact persistent configuration depends on the distribution.

### NetworkManager

Check OVS support:

```shell
nmcli connection show
nmcli connection add type ovs-bridge conn.interface br-provider con-name br-provider
nmcli connection add type ovs-port conn.interface bond-provider master br-provider con-name bond-provider
nmcli connection add type ethernet conn.interface eno1 master bond-provider con-name eno1-ovs
nmcli connection add type ethernet conn.interface eno2 master bond-provider con-name eno2-ovs
```

Then set bond options:

```shell
nmcli connection modify bond-provider ovs-port.bond-mode active-backup
nmcli connection up br-provider
nmcli connection up bond-provider
nmcli connection up eno1-ovs
nmcli connection up eno2-ovs
```

NetworkManager OVS properties vary by distribution version. Confirm with:

```shell
nmcli connection show bond-provider
```

### systemd-networkd

When using systemd-networkd, create persistent `.netdev` and `.network` files
for the OVS bridge and ports only if your distribution supports OVS integration.
Many environments still use NetworkManager, ifupdown, ifcfg scripts, or custom
systemd units for OVS.

### Simple systemd restore unit

For a lab, use an explicit OVS restore script after `openvswitch` starts.

Example command sequence:

```shell
sudo ovs-vsctl --may-exist add-br br-provider
sudo ovs-vsctl --may-exist add-bond br-provider bond-provider eno1 eno2 bond_mode=active-backup other_config:bond-primary=eno1
sudo ip link set br-provider up
sudo ip link set eno1 up
sudo ip link set eno2 up
```

For production, prefer the distribution network manager so IP addresses, routes,
DNS, and OVS objects are managed together.

## 8. Attach KVM VMs To The HA Bridge

Libvirt interface XML:

```xml
<interface type='bridge'>
  <source bridge='br-provider'/>
  <virtualport type='openvswitch'/>
  <model type='virtio'/>
</interface>
```

Attach an existing VM:

```shell
virsh attach-interface VM_NAME \
  --type bridge \
  --source br-provider \
  --model virtio \
  --config \
  --live
```

If the VM must use a VLAN, set the tag on the VM port after the tap exists:

```shell
sudo ovs-vsctl set port vnet3 tag=1927
```

Verify:

```shell
sudo ovs-vsctl show
sudo ovs-vsctl list port vnet3
```

## 9. Use OVS HA With OVN Provider Networks

OVN usually uses `br-int` for logical ports. Provider network traffic can exit
through a physical OVS bridge such as `br-provider`.

Set a bridge mapping on each KVM host:

```shell
sudo ovs-vsctl set open . external_ids:ovn-bridge-mappings=provider:br-provider
```

Create the localnet port in OVN:

```shell
sudo ovn-nbctl ls-add provider-net
sudo ovn-nbctl lsp-add provider-net provider-localnet
sudo ovn-nbctl lsp-set-type provider-localnet localnet
sudo ovn-nbctl lsp-set-addresses provider-localnet unknown
sudo ovn-nbctl lsp-set-options provider-localnet network_name=provider
```

If the provider network uses VLAN `1927`:

```shell
sudo ovn-nbctl set logical_switch provider-net other_config:subnet=192.168.27.0/24
sudo ovn-nbctl set logical_switch_port provider-localnet tag=1927
```

Verify on the KVM host:

```shell
sudo ovs-vsctl get open . external_ids:ovn-bridge-mappings
sudo ovs-vsctl show
```

## 10. Test Failover

Start continuous traffic from a VM or the host:

```shell
ping -i 0.2 192.168.100.1
```

Watch the bond:

```shell
watch -n 1 "sudo ovs-appctl bond/show bond-provider"
```

Disconnect or disable the active link:

```shell
sudo ip link set eno1 down
```

Expected:

- Bond active member changes to `eno2`.
- Ping may lose a small number of packets.
- VM traffic resumes through the standby link.

Restore:

```shell
sudo ip link set eno1 up
```

Test the other member:

```shell
sudo ip link set eno2 down
sudo ip link set eno2 up
```

For LACP, also check:

```shell
sudo ovs-appctl lacp/show bond-provider
```

## 11. Monitor OVS HA

Useful commands:

```shell
sudo ovs-vsctl show
sudo ovs-vsctl list bridge br-provider
sudo ovs-vsctl list port bond-provider
sudo ovs-vsctl list interface eno1
sudo ovs-vsctl list interface eno2
sudo ovs-appctl bond/show bond-provider
sudo ovs-appctl lacp/show bond-provider
sudo ovs-ofctl show br-provider
sudo ip -s link show eno1
sudo ip -s link show eno2
```

Check logs:

```shell
sudo journalctl -u openvswitch-switch -n 200 --no-pager
sudo journalctl -u ovs-vswitchd -n 200 --no-pager
sudo journalctl -u ovsdb-server -n 200 --no-pager
```

Service names vary by distribution.

## 12. Troubleshooting

### Bond Does Not Fail Over

Check member state:

```shell
sudo ovs-appctl bond/show bond-provider
ip -br link show eno1 eno2
```

Common causes:

- Physical link still appears up but upstream switch path is broken.
- Switch blocks one port due to loop protection.
- LACP mismatch.
- NIC driver issue.
- Firewall or VLAN mismatch on the standby path.

### LACP Is Not Attached

Check:

```shell
sudo ovs-appctl lacp/show bond-provider
```

Common causes:

- Switch port-channel is not configured.
- One side is static while OVS expects LACP.
- Links connect to two independent switches without MLAG.
- VLAN trunk configuration differs between member ports.

### VM Cannot Reach External Network

Check VM tap port:

```shell
sudo ovs-vsctl show
sudo ovs-vsctl list port vnet3
```

Check VLAN:

```shell
sudo ovs-vsctl get port vnet3 tag
sudo ovs-vsctl get port bond-provider trunks
```

Check physical traffic:

```shell
sudo tcpdump -ni eno1 vlan
sudo tcpdump -ni eno2 vlan
```

### Host Lost Management Access

Use console access and check:

```shell
ip -br addr
ip route
sudo ovs-vsctl show
sudo ovs-appctl bond/show bond-provider
```

Common causes:

- Management IP remained on a physical NIC that was added to OVS.
- Default route was not moved to the OVS bridge or management VLAN interface.
- VLAN tag is missing or wrong.
- Upstream switch trunk does not allow the management VLAN.

## 13. Best Practices

- Use at least two physical NICs per KVM host for production traffic.
- Connect redundant NICs to redundant switches.
- Use `active-backup` when switch-side LACP or MLAG is not available.
- Use `balance-tcp` with `lacp=active` only with correct switch-side LACP.
- Keep management access out-of-band when changing bridge or bond settings.
- Test failover before running production VMs.
- Keep OVS bridge names consistent across KVM hosts.
- Document VLANs, trunks, native VLANs, and bridge mappings.
- Monitor OVS bond state and physical NIC errors.

## Summary

OVS HA for KVM is mainly about making each hypervisor's network path resilient.
The common pattern is:

```text
physical NICs -> OVS bond -> OVS bridge -> VM ports or OVN provider bridge
```

Use `active-backup` for simple redundancy, use LACP only when the switching
fabric is designed for it, and always validate failover with real traffic.
