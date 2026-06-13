---
tags:
  - kvm
  - ovn
  - openvswitch
  - ha
---

# OVN HA For KVM

## Overview

OVN has two main parts:

- OVN central control plane: `ovn-nbdb`, `ovn-sbdb`, and `ovn-northd`
- OVN host data plane: `ovs-vswitchd`, `ovsdb-server`, and `ovn-controller`

For HA, protect both layers:

- Run OVN Northbound and Southbound databases as 3-node clustered databases.
- Run `ovn-northd` on all central nodes. Only one instance is active at a time.
- Configure every KVM host with multiple Southbound database endpoints.
- Use redundant underlay networking for Geneve traffic.
- Keep VM networking on `br-int`; `ovn-controller` programs flows locally on each KVM host.

```text
                 OVN Central Cluster
       +----------------+----------------+
       |                |                |
  central-a        central-b        central-c
  NB/SB DB         NB/SB DB         NB/SB DB
  ovn-northd       ovn-northd       ovn-northd
       |                |                |
       +------- clustered DB raft -------+
                        |
             SB remote list on each host
                        |
      +-----------------+-----------------+
      |                 |                 |
  kvm-host-a        kvm-host-b        kvm-host-c
  OVS + OVN         OVS + OVN         OVS + OVN
  br-int            br-int            br-int
```

## Lab Assumptions

| Node | Role | Underlay IP |
|---|---|---:|
| `central-a` | OVN central node 1 | `192.168.100.10` |
| `central-b` | OVN central node 2 | `192.168.100.20` |
| `central-c` | OVN central node 3 | `192.168.100.30` |
| `kvm-host-a` | KVM + OVS + OVN controller | `192.168.100.11` |
| `kvm-host-b` | KVM + OVS + OVN controller | `192.168.100.12` |
| `kvm-host-c` | KVM + OVS + OVN controller | `192.168.100.13` |

OVN database ports:

| Port | Service |
|---:|---|
| `6641/tcp` | OVN Northbound database |
| `6642/tcp` | OVN Southbound database |
| `6081/udp` | Geneve tunnel traffic between chassis |

## 1. Install Packages

Install OVN central packages on `central-a`, `central-b`, and `central-c`.

Ubuntu/Debian:

```shell
sudo apt-get update
sudo apt-get install -y ovn-central ovn-common openvswitch-switch
```

RHEL/CentOS/Rocky:

```shell
sudo dnf install -y ovn-central ovn openvswitch
```

Install OVN host packages on every KVM host.

Ubuntu/Debian:

```shell
sudo apt-get update
sudo apt-get install -y ovn-host ovn-common openvswitch-switch libvirt-daemon-system
```

RHEL/CentOS/Rocky:

```shell
sudo dnf install -y ovn-host ovn openvswitch libvirt
```

Enable Open vSwitch on every node:

```shell
sudo systemctl enable --now openvswitch-switch || sudo systemctl enable --now openvswitch
```

## 2. Prepare Firewall Rules

On all central nodes, allow OVN database traffic from other central nodes and KVM
hosts.

```shell
sudo firewall-cmd --add-port=6641/tcp --permanent
sudo firewall-cmd --add-port=6642/tcp --permanent
sudo firewall-cmd --reload
```

On every KVM host, allow Geneve between hypervisors:

```shell
sudo firewall-cmd --add-port=6081/udp --permanent
sudo firewall-cmd --reload
```

For production, restrict these rules to the management and tunnel networks.

## 3. Create The First OVN Central Cluster Node

Run on `central-a`.

Stop OVN central before rebuilding the database files:

```shell
sudo systemctl stop ovn-central
```

Create clustered Northbound and Southbound databases:

```shell
sudo ovn-ctl \
  --db-nb-create-insecure-remote=yes \
  --db-sb-create-insecure-remote=yes \
  --db-nb-cluster-local-addr=192.168.100.10 \
  --db-sb-cluster-local-addr=192.168.100.10 \
  start_northd
```

Start the service:

```shell
sudo systemctl enable --now ovn-central
```

Verify listeners:

```shell
sudo ss -lntp | grep -E '6641|6642'
```

Expected:

```text
6641 northbound database
6642 southbound database
```

## 4. Join The Second Central Node

Run on `central-b`.

```shell
sudo systemctl stop ovn-central
```

Join `central-b` to the cluster through `central-a`:

```shell
sudo ovn-ctl \
  --db-nb-create-insecure-remote=yes \
  --db-sb-create-insecure-remote=yes \
  --db-nb-cluster-local-addr=192.168.100.20 \
  --db-sb-cluster-local-addr=192.168.100.20 \
  --db-nb-cluster-remote-addr=192.168.100.10 \
  --db-sb-cluster-remote-addr=192.168.100.10 \
  start_northd
```

Start the service:

```shell
sudo systemctl enable --now ovn-central
```

## 5. Join The Third Central Node

Run on `central-c`.

```shell
sudo systemctl stop ovn-central
```

Join `central-c` to the cluster:

```shell
sudo ovn-ctl \
  --db-nb-create-insecure-remote=yes \
  --db-sb-create-insecure-remote=yes \
  --db-nb-cluster-local-addr=192.168.100.30 \
  --db-sb-cluster-local-addr=192.168.100.30 \
  --db-nb-cluster-remote-addr=192.168.100.10 \
  --db-sb-cluster-remote-addr=192.168.100.10 \
  start_northd
```

Start the service:

```shell
sudo systemctl enable --now ovn-central
```

## 6. Verify Cluster Health

Run on any central node.

Check the Northbound database:

```shell
sudo ovs-appctl -t /var/run/ovn/ovnnb_db.ctl cluster/status OVN_Northbound
```

Check the Southbound database:

```shell
sudo ovs-appctl -t /var/run/ovn/ovnsb_db.ctl cluster/status OVN_Southbound
```

Healthy output should show:

- `Status: cluster member`
- One leader
- Other nodes connected as followers
- No election loop

Check OVN from client tools:

```shell
sudo ovn-nbctl --db=tcp:192.168.100.10:6641,tcp:192.168.100.20:6641,tcp:192.168.100.30:6641 show
sudo ovn-sbctl --db=tcp:192.168.100.10:6642,tcp:192.168.100.20:6642,tcp:192.168.100.30:6642 show
```

## 7. Configure KVM Hosts To Use All SB Databases

Run on every KVM host. Use the host's own underlay IP as `ovn-encap-ip`.

On `kvm-host-a`:

```shell
sudo ovs-vsctl set open . external_ids:ovn-remote="tcp:192.168.100.10:6642,tcp:192.168.100.20:6642,tcp:192.168.100.30:6642"
sudo ovs-vsctl set open . external_ids:ovn-encap-type=geneve
sudo ovs-vsctl set open . external_ids:ovn-encap-ip=192.168.100.11
sudo ovs-vsctl --may-exist add-br br-int
sudo systemctl enable --now ovn-host
```

On `kvm-host-b`:

```shell
sudo ovs-vsctl set open . external_ids:ovn-remote="tcp:192.168.100.10:6642,tcp:192.168.100.20:6642,tcp:192.168.100.30:6642"
sudo ovs-vsctl set open . external_ids:ovn-encap-type=geneve
sudo ovs-vsctl set open . external_ids:ovn-encap-ip=192.168.100.12
sudo ovs-vsctl --may-exist add-br br-int
sudo systemctl enable --now ovn-host
```

On `kvm-host-c`:

```shell
sudo ovs-vsctl set open . external_ids:ovn-remote="tcp:192.168.100.10:6642,tcp:192.168.100.20:6642,tcp:192.168.100.30:6642"
sudo ovs-vsctl set open . external_ids:ovn-encap-type=geneve
sudo ovs-vsctl set open . external_ids:ovn-encap-ip=192.168.100.13
sudo ovs-vsctl --may-exist add-br br-int
sudo systemctl enable --now ovn-host
```

Verify each chassis registered:

```shell
sudo ovn-sbctl --db=tcp:192.168.100.10:6642,tcp:192.168.100.20:6642,tcp:192.168.100.30:6642 show
```

Expected:

```text
Chassis kvm-host-a
Chassis kvm-host-b
Chassis kvm-host-c
```

## 8. Configure Northbound Client Access

For admin commands, always pass all Northbound endpoints:

```shell
sudo ovn-nbctl --db=tcp:192.168.100.10:6641,tcp:192.168.100.20:6641,tcp:192.168.100.30:6641 show
```

Optional shell helper:

```shell
export OVN_NB_DB=tcp:192.168.100.10:6641,tcp:192.168.100.20:6641,tcp:192.168.100.30:6641
export OVN_SB_DB=tcp:192.168.100.10:6642,tcp:192.168.100.20:6642,tcp:192.168.100.30:6642
```

Then:

```shell
ovn-nbctl show
ovn-sbctl show
```

## 9. Create A Test Logical Network

Run from any central node or admin host with access to the NB database.

```shell
sudo ovn-nbctl --db=$OVN_NB_DB ls-add ls-ha

sudo ovn-nbctl --db=$OVN_NB_DB lsp-add ls-ha vm-a-port
sudo ovn-nbctl --db=$OVN_NB_DB lsp-set-addresses vm-a-port "02:00:00:00:10:11 10.40.0.11"

sudo ovn-nbctl --db=$OVN_NB_DB lsp-add ls-ha vm-b-port
sudo ovn-nbctl --db=$OVN_NB_DB lsp-set-addresses vm-b-port "02:00:00:00:10:12 10.40.0.12"
```

Attach VM tap interfaces to `br-int` and set the logical port ID on the OVS
interface.

Example on `kvm-host-a` after the VM tap exists:

```shell
sudo ovs-vsctl set Interface vnet0 external_ids:iface-id=vm-a-port
```

Example on `kvm-host-b`:

```shell
sudo ovs-vsctl set Interface vnet0 external_ids:iface-id=vm-b-port
```

Check binding:

```shell
sudo ovn-sbctl --db=$OVN_SB_DB list Port_Binding vm-a-port
sudo ovn-sbctl --db=$OVN_SB_DB list Port_Binding vm-b-port
```

## 10. Test OVN Central Failover

Start a continuous ping between VMs on different KVM hosts.

Stop one follower central node:

```shell
sudo systemctl stop ovn-central
```

Expected result:

- Existing VM traffic continues.
- `ovn-controller` stays connected through another SB database endpoint.
- New OVN changes still work if the cluster has quorum.

Stop the current leader and watch a new leader election:

```shell
sudo ovs-appctl -t /var/run/ovn/ovnnb_db.ctl cluster/status OVN_Northbound
sudo systemctl stop ovn-central
```

Check status from another central node:

```shell
sudo ovs-appctl -t /var/run/ovn/ovnnb_db.ctl cluster/status OVN_Northbound
sudo ovs-appctl -t /var/run/ovn/ovnsb_db.ctl cluster/status OVN_Southbound
```

Expected result:

- A new leader is elected.
- Admin commands continue when using the multi-endpoint DB string.
- VM data-plane traffic should not depend on a live central node after flows are programmed.

Restart the stopped node:

```shell
sudo systemctl start ovn-central
```

## 11. Quorum Rules

Use an odd number of central nodes.

| Central nodes | Can tolerate failures | Quorum required |
|---:|---:|---:|
| 1 | 0 | 1 |
| 3 | 1 | 2 |
| 5 | 2 | 3 |

Do not run a 2-node OVN central cluster for production. If one node fails or the
link between nodes breaks, the cluster can lose quorum.

## 12. Use TLS Instead Of Insecure TCP

The examples above use plain TCP because it is simple for a lab. In production,
use SSL/TLS for NB and SB databases.

Typical production requirements:

- Private CA for OVN certificates
- Server certificates on central nodes
- Client certificates for `ovn-controller` and admin hosts
- Firewall rules restricted to trusted management networks

Example connection format:

```shell
ssl:192.168.100.10:6642,ssl:192.168.100.20:6642,ssl:192.168.100.30:6642
```

## 13. Operational Commands

Show central cluster status:

```shell
sudo ovs-appctl -t /var/run/ovn/ovnnb_db.ctl cluster/status OVN_Northbound
sudo ovs-appctl -t /var/run/ovn/ovnsb_db.ctl cluster/status OVN_Southbound
```

Show OVN chassis:

```shell
sudo ovn-sbctl --db=$OVN_SB_DB show
```

Show host OVN remote configuration:

```shell
sudo ovs-vsctl get open . external_ids:ovn-remote
sudo ovs-vsctl get open . external_ids:ovn-encap-ip
sudo ovs-vsctl get open . external_ids:ovn-encap-type
```

Check `ovn-controller` status:

```shell
sudo systemctl status ovn-host
sudo journalctl -u ovn-host -n 100 --no-pager
```

Check OVS and OVN local state:

```shell
sudo ovs-vsctl show
sudo ovs-vsctl list open_vswitch
sudo ovs-ofctl dump-flows br-int
```

Check tunnel traffic:

```shell
sudo tcpdump -ni any udp port 6081
```

## 14. Troubleshooting

### Chassis Does Not Appear

Check the host remote:

```shell
sudo ovs-vsctl get open . external_ids:ovn-remote
```

Check connectivity to all SB endpoints:

```shell
nc -vz 192.168.100.10 6642
nc -vz 192.168.100.20 6642
nc -vz 192.168.100.30 6642
```

Check `ovn-controller` logs:

```shell
sudo journalctl -u ovn-host -n 200 --no-pager
```

### VM Port Is Not Bound

Check the logical switch port:

```shell
sudo ovn-nbctl --db=$OVN_NB_DB lsp-list ls-ha
```

Check the OVS interface external ID:

```shell
sudo ovs-vsctl list Interface vnet0
```

The OVS interface must have:

```text
external_ids:iface-id=vm-a-port
```

Check the Southbound binding:

```shell
sudo ovn-sbctl --db=$OVN_SB_DB list Port_Binding vm-a-port
```

### Cluster Has No Leader

Check cluster status on all central nodes:

```shell
sudo ovs-appctl -t /var/run/ovn/ovnnb_db.ctl cluster/status OVN_Northbound
sudo ovs-appctl -t /var/run/ovn/ovnsb_db.ctl cluster/status OVN_Southbound
```

Common causes:

- Fewer than quorum nodes are online.
- Central nodes cannot reach each other.
- Firewall blocks database cluster traffic.
- Time drift is severe.
- Nodes were initialized as separate clusters instead of joining the first node.

## Summary

OVN HA is mainly control-plane HA:

- Use 3 OVN central nodes.
- Cluster both NB and SB databases.
- Run `ovn-northd` on every central node.
- Configure every KVM host with all SB database endpoints.
- Keep Geneve underlay networking redundant and reachable.
- Use TLS and restricted firewall rules in production.

The data plane is distributed. Once `ovn-controller` has programmed OVS flows,
existing VM traffic can continue during a temporary OVN central outage. New
logical network changes require OVN central quorum.
