---
tags:
  - linux
  - network
  - iproute2
---

# Linux `ip` command

`ip` is the main command-line tool from the `iproute2` package. It is used to
inspect and manage Linux network interfaces, IP addresses, routes, ARP/NDP
neighbors, network namespaces, tunnels, and policy routing.

Modern Linux systems should prefer `ip` over older commands such as
`ifconfig`, `route`, `arp`, and parts of `netstat`.

## Why migrate to `ip`

### 1. Old net-tools are deprecated

Traditional tools come from the `net-tools` package:

| Old tool | Modern replacement |
| --- | --- |
| `ifconfig` | `ip addr`, `ip link` |
| `route` | `ip route` |
| `arp` | `ip neigh` |
| `netstat -r` | `ip route` |
| `netstat -i` | `ip -s link` |

Many distributions no longer install `net-tools` by default. `iproute2` is the
standard Linux networking toolset and is maintained for modern kernel features.

### 2. `ip` maps better to the Linux kernel networking model

Linux networking is broader than simple interface and route management. Modern
systems often use:

- Multiple routing tables
- Policy-based routing
- VLANs
- Bridges
- Bonds
- Tunnels
- VRF
- Network namespaces
- IPv6
- Containers and virtual Ethernet devices

`ifconfig` and `route` were designed for simpler networking models. `ip`
understands the current kernel networking stack much better.

### 3. Better output and automation

`ip` has consistent subcommands:

```bash
ip link
ip addr
ip route
ip neigh
ip rule
ip netns
```

It also supports compact output and JSON output:

```bash
ip -br addr
ip -j addr
ip -j route
```

This makes it easier to use in scripts and troubleshooting tools.

### 4. Strong IPv6 support

IPv6 is first-class in `ip`:

```bash
ip -6 addr
ip -6 route
ip -6 neigh
```

With older tools, IPv6 support is often incomplete or requires separate
commands.

## Basic syntax

General form:

```bash
ip [OPTIONS] OBJECT COMMAND
```

Common objects:

| Object | Purpose |
| --- | --- |
| `link` | Network interfaces at layer 2 |
| `addr` | IPv4/IPv6 addresses |
| `route` | Routing table |
| `neigh` | ARP/NDP neighbor table |
| `rule` | Policy routing rules |
| `netns` | Network namespaces |
| `tunnel` | IP tunnels |
| `maddr` | Multicast addresses |

Useful options:

| Option | Meaning |
| --- | --- |
| `-br` | Brief output |
| `-c` | Colored output |
| `-s` | Statistics |
| `-d` | Detailed output |
| `-j` | JSON output |
| `-4` | IPv4 only |
| `-6` | IPv6 only |

Examples:

```bash
ip -br addr
ip -c route
ip -s link show eth0
ip -j addr show eth0
```

## Interface management with `ip link`

Show all network interfaces:

```bash
ip link show
```

Brief output:

```bash
ip -br link
```

Show one interface:

```bash
ip link show dev eth0
```

Bring an interface up:

```bash
sudo ip link set dev eth0 up
```

Bring an interface down:

```bash
sudo ip link set dev eth0 down
```

Change MTU:

```bash
sudo ip link set dev eth0 mtu 9000
```

Rename an interface:

```bash
sudo ip link set dev eth0 name lan0
```

Show interface counters:

```bash
ip -s link show dev eth0
```

Example interpretation:

```text
RX: bytes  packets  errors  dropped
TX: bytes  packets  errors  dropped
```

If `errors` or `dropped` keeps increasing, check physical links, duplex/speed,
driver issues, overloaded queues, firewall drops, or MTU mismatch.

## IP address management with `ip addr`

Show all addresses:

```bash
ip addr show
```

Short form:

```bash
ip a
```

Brief output:

```bash
ip -br addr
```

Show only IPv4 addresses:

```bash
ip -4 addr
```

Show only IPv6 addresses:

```bash
ip -6 addr
```

Show addresses on one interface:

```bash
ip addr show dev eth0
```

Add an IPv4 address:

```bash
sudo ip addr add 192.168.10.20/24 dev eth0
```

Delete an IPv4 address:

```bash
sudo ip addr del 192.168.10.20/24 dev eth0
```

Add an IPv6 address:

```bash
sudo ip -6 addr add 2001:db8:10::20/64 dev eth0
```

Flush all addresses from an interface:

```bash
sudo ip addr flush dev eth0
```

Important: changes made by `ip addr add`, `ip addr del`, and `ip link set` are
runtime changes. They usually disappear after reboot or network service restart.
For permanent configuration, use the distribution network manager, such as
NetworkManager, systemd-networkd, Netplan, or ifcfg files.

## Routing with `ip route`

Show routes:

```bash
ip route show
```

Short form:

```bash
ip r
```

Show IPv6 routes:

```bash
ip -6 route
```

Show the route used for one destination:

```bash
ip route get 8.8.8.8
```

Example output:

```text
8.8.8.8 via 192.168.10.1 dev eth0 src 192.168.10.20
```

Meaning:

| Field | Meaning |
| --- | --- |
| `via 192.168.10.1` | Next-hop gateway |
| `dev eth0` | Outgoing interface |
| `src 192.168.10.20` | Source IP selected by the kernel |

Add a default gateway:

```bash
sudo ip route add default via 192.168.10.1 dev eth0
```

Replace the default gateway:

```bash
sudo ip route replace default via 192.168.10.1 dev eth0
```

Delete the default gateway:

```bash
sudo ip route del default
```

Add a static route:

```bash
sudo ip route add 10.20.0.0/16 via 192.168.10.254 dev eth0
```

Delete a static route:

```bash
sudo ip route del 10.20.0.0/16
```

Add a direct route through an interface:

```bash
sudo ip route add 172.16.1.0/24 dev eth1
```

## Neighbor table with `ip neigh`

`ip neigh` replaces `arp` for IPv4 and also works with IPv6 NDP.

Show neighbor entries:

```bash
ip neigh show
```

Show entries for one interface:

```bash
ip neigh show dev eth0
```

Delete one neighbor entry:

```bash
sudo ip neigh del 192.168.10.1 dev eth0
```

Flush neighbor entries on one interface:

```bash
sudo ip neigh flush dev eth0
```

Common states:

| State | Meaning |
| --- | --- |
| `REACHABLE` | Recently confirmed reachable |
| `STALE` | Known, but not recently verified |
| `DELAY` | Waiting before probing |
| `PROBE` | Actively checking reachability |
| `FAILED` | Resolution failed |
| `PERMANENT` | Static neighbor entry |

Example troubleshooting:

```bash
ip neigh show 192.168.10.1
```

If the gateway is `FAILED`, the host cannot resolve the gateway MAC address.
Check VLAN, subnet mask, switch port, cable, gateway availability, or duplicate
IP problems.

## Policy routing with `ip rule`

Normal routing uses the main routing table. Policy routing lets Linux choose a
route table based on source address, destination address, firewall mark, input
interface, or other selectors.

Show rules:

```bash
ip rule show
```

Show a custom table:

```bash
ip route show table 100
```

Example: traffic from `192.168.20.10` uses routing table `100`.

```bash
sudo ip route add default via 192.168.20.1 dev eth1 table 100
sudo ip rule add from 192.168.20.10/32 table 100
```

Check route selection:

```bash
ip route get 8.8.8.8 from 192.168.20.10
```

Delete the rule:

```bash
sudo ip rule del from 192.168.20.10/32 table 100
```

Policy routing is common on multi-homed servers, VPN gateways, routers,
containers, and hosts with separate management and service networks.

## Network namespaces with `ip netns`

Network namespaces provide isolated network stacks. Each namespace has its own
interfaces, addresses, routes, firewall rules, and neighbor table.

List namespaces:

```bash
ip netns list
```

Create a namespace:

```bash
sudo ip netns add ns1
```

Run a command inside a namespace:

```bash
sudo ip netns exec ns1 ip addr
```

Create a veth pair:

```bash
sudo ip link add veth-host type veth peer name veth-ns
```

Move one end into the namespace:

```bash
sudo ip link set veth-ns netns ns1
```

Configure the host side:

```bash
sudo ip addr add 10.10.10.1/24 dev veth-host
sudo ip link set veth-host up
```

Configure the namespace side:

```bash
sudo ip netns exec ns1 ip addr add 10.10.10.2/24 dev veth-ns
sudo ip netns exec ns1 ip link set veth-ns up
sudo ip netns exec ns1 ip link set lo up
```

Test connectivity:

```bash
ping -c 3 10.10.10.2
sudo ip netns exec ns1 ping -c 3 10.10.10.1
```

Delete the namespace:

```bash
sudo ip netns del ns1
```

This model is the base idea behind container networking.

## Practical examples

### Example 1: Check basic host networking

```bash
ip -br link
ip -br addr
ip route
ip route get 8.8.8.8
ip neigh
```

What to verify:

- The interface is `UP`
- The host has the expected IP address and prefix
- A default route exists
- The selected source IP is correct
- The gateway appears in the neighbor table

### Example 2: Temporarily assign a test IP

```bash
sudo ip addr add 192.168.50.10/24 dev eth0
sudo ip link set eth0 up
ip -br addr show eth0
```

Test:

```bash
ping -c 3 192.168.50.1
```

Remove it:

```bash
sudo ip addr del 192.168.50.10/24 dev eth0
```

Use this when testing an address before making a permanent configuration
change.

### Example 3: Add a temporary route to a remote network

Assume:

- Local interface: `eth0`
- Local subnet: `192.168.10.0/24`
- Gateway to remote network: `192.168.10.254`
- Remote network: `10.30.0.0/16`

Add route:

```bash
sudo ip route add 10.30.0.0/16 via 192.168.10.254 dev eth0
```

Verify:

```bash
ip route get 10.30.1.20
```

Delete route:

```bash
sudo ip route del 10.30.0.0/16
```

### Example 4: Find why traffic uses the wrong interface

Show the route decision:

```bash
ip route get 1.1.1.1
```

If the output says:

```text
1.1.1.1 via 10.0.0.1 dev eth1 src 10.0.0.20
```

Then Linux will send the packet through `eth1` using source IP `10.0.0.20`.
Check default routes and route priorities:

```bash
ip route
```

Routes with lower metric are preferred:

```text
default via 10.0.0.1 dev eth1 metric 100
default via 192.168.10.1 dev eth0 metric 200
```

In this case, `eth1` wins because metric `100` is lower than `200`.

### Example 5: Replace `ifconfig`

Old command:

```bash
ifconfig eth0
```

Modern command:

```bash
ip addr show dev eth0
ip -s link show dev eth0
```

Old command:

```bash
ifconfig eth0 up
```

Modern command:

```bash
sudo ip link set dev eth0 up
```

Old command:

```bash
ifconfig eth0 192.168.10.20 netmask 255.255.255.0
```

Modern command:

```bash
sudo ip addr add 192.168.10.20/24 dev eth0
```

### Example 6: Replace `route`

Old command:

```bash
route -n
```

Modern command:

```bash
ip route
```

Old command:

```bash
route add default gw 192.168.10.1
```

Modern command:

```bash
sudo ip route add default via 192.168.10.1
```

Old command:

```bash
route add -net 10.20.0.0 netmask 255.255.0.0 gw 192.168.10.254
```

Modern command:

```bash
sudo ip route add 10.20.0.0/16 via 192.168.10.254
```

### Example 7: Replace `arp`

Old command:

```bash
arp -n
```

Modern command:

```bash
ip neigh
```

Old command:

```bash
arp -d 192.168.10.1
```

Modern command:

```bash
sudo ip neigh del 192.168.10.1 dev eth0
```

## Troubleshooting workflow

When a Linux host cannot reach the network, use this order:

### 1. Check link state

```bash
ip -br link
ip -s link show dev eth0
```

Look for:

- `UP` or `DOWN`
- Increasing `RX/TX` counters
- Increasing errors or drops

### 2. Check IP address

```bash
ip -br addr show eth0
```

Confirm:

- Correct IP address
- Correct prefix length, such as `/24`
- No unexpected duplicate or stale addresses

### 3. Check routes

```bash
ip route
ip route get 8.8.8.8
```

Confirm:

- Default route exists
- Gateway is correct
- Outgoing interface is correct
- Source IP is correct

### 4. Check neighbor resolution

```bash
ip neigh show dev eth0
```

Confirm:

- Gateway has a MAC address
- State is not always `FAILED`

### 5. Check policy routing

```bash
ip rule
ip route show table all
```

This is important on hosts with multiple NICs, VPNs, containers, or advanced
routing.

## Common mistakes

### Missing prefix length

Wrong:

```bash
sudo ip addr add 192.168.10.20 dev eth0
```

Better:

```bash
sudo ip addr add 192.168.10.20/24 dev eth0
```

Without the correct prefix length, the kernel may create an unexpected route.

### Adding duplicate default routes

Check first:

```bash
ip route
```

Use `replace` when the intent is to change the default gateway:

```bash
sudo ip route replace default via 192.168.10.1 dev eth0
```

### Forgetting that `ip` changes are temporary

Most `ip` commands change runtime kernel state only. After reboot, the
configuration may be lost. Use persistent network configuration for production
changes.

### Confusing interface state with cable state

`ip link set eth0 up` enables the interface administratively. It does not
guarantee physical carrier.

Check detailed state:

```bash
ip -d link show dev eth0
```

Look for fields such as `state UP`, `LOWER_UP`, or `NO-CARRIER`.

## Quick reference

| Task | Command |
| --- | --- |
| Show interfaces | `ip -br link` |
| Show addresses | `ip -br addr` |
| Show one interface | `ip addr show dev eth0` |
| Bring interface up | `sudo ip link set eth0 up` |
| Bring interface down | `sudo ip link set eth0 down` |
| Add address | `sudo ip addr add 192.168.10.20/24 dev eth0` |
| Delete address | `sudo ip addr del 192.168.10.20/24 dev eth0` |
| Show routes | `ip route` |
| Show route decision | `ip route get 8.8.8.8` |
| Add default route | `sudo ip route add default via 192.168.10.1` |
| Replace default route | `sudo ip route replace default via 192.168.10.1` |
| Delete default route | `sudo ip route del default` |
| Show neighbors | `ip neigh` |
| Flush neighbors | `sudo ip neigh flush dev eth0` |
| Show policy rules | `ip rule` |
| Show namespaces | `ip netns list` |

## Summary

Migrate to `ip` because it is the standard Linux networking tool, supports
modern kernel features, works well with IPv4 and IPv6, and provides better
output for troubleshooting and automation.

For daily operations, remember these core commands:

```bash
ip -br link
ip -br addr
ip route
ip route get <destination>
ip neigh
ip rule
```

These commands answer the most important network questions:

- Is the interface up?
- Does it have the right address?
- Which route will Linux use?
- Which source IP will Linux choose?
- Can the host resolve the gateway MAC address?
- Are policy routing rules changing the route decision?
