---
tags:
  - HA-VPN
  - GCP
  - google-cloud
  - cloud
  - google
---
# GCP HA VPN Lab: Connect Two VPC Networks

This lab creates two custom VPC networks in one Google Cloud project and connects them with **HA VPN**. It simulates a real site-to-site VPN design by using:

- Two VPC networks: `vpc-a` and `vpc-b`
- One HA VPN gateway in each VPC
- One Cloud Router in each VPC
- Four VPN tunnels total, two from each gateway
- BGP dynamic routing over the VPN tunnels
- Two test VMs to verify private IP connectivity

> Official reference: [Create HA VPN gateways to connect VPC networks](https://cloud.google.com/network-connectivity/docs/vpn/how-to/creating-ha-vpn2)

## Architecture

```text
vpc-a / subnet-a                       vpc-b / subnet-b
10.10.0.0/24                           10.20.0.0/24

vm-a                                   vm-b
  |                                      |
Cloud Router A                       Cloud Router B
ASN 65001                            ASN 65002
  |                                      |
HA VPN Gateway A  <==== 4 tunnels ====> HA VPN Gateway B
```

For the 99.99% HA VPN SLA, Google Cloud requires one tunnel on each interface of each HA VPN gateway. When connecting two Google Cloud VPC networks, both HA VPN gateways must be in the same region, and interface `0` connects to interface `0`, while interface `1` connects to interface `1`.

## Prerequisites

Run these commands from **Google Cloud Shell** or any machine with the Google Cloud CLI authenticated.

Required IAM role:

- `roles/compute.networkAdmin`

Enable Compute Engine API:

```shell
gcloud services enable compute.googleapis.com
```

Set your project:

```shell
gcloud config set project PROJECT_ID
gcloud config list --format='text(core.project)'
```

## 1. Set Lab Variables

```shell
export PROJECT_ID="$(gcloud config get-value project)"
export REGION="us-central1"
export ZONE="us-central1-a"

export VPC_A="vpc-a"
export VPC_B="vpc-b"
export SUBNET_A="subnet-a"
export SUBNET_B="subnet-b"
export CIDR_A="10.10.0.0/24"
export CIDR_B="10.20.0.0/24"

export GW_A="ha-vpn-gw-a"
export GW_B="ha-vpn-gw-b"
export ROUTER_A="router-a"
export ROUTER_B="router-b"
export ASN_A="65001"
export ASN_B="65002"

export TUNNEL_A_TO_B_0="tunnel-a-to-b-if-0"
export TUNNEL_A_TO_B_1="tunnel-a-to-b-if-1"
export TUNNEL_B_TO_A_0="tunnel-b-to-a-if-0"
export TUNNEL_B_TO_A_1="tunnel-b-to-a-if-1"

export SHARED_SECRET="$(openssl rand -base64 32)"

gcloud config set compute/region "$REGION"
gcloud config set compute/zone "$ZONE"
```

## 2. Create Two Custom VPC Networks

Use custom mode VPCs so you fully control subnet ranges. The subnet ranges must not overlap.

```shell
gcloud compute networks create "$VPC_A" \
  --subnet-mode=custom \
  --bgp-routing-mode=global

gcloud compute networks create "$VPC_B" \
  --subnet-mode=custom \
  --bgp-routing-mode=global
```

Create one subnet in each VPC:

```shell
gcloud compute networks subnets create "$SUBNET_A" \
  --network="$VPC_A" \
  --region="$REGION" \
  --range="$CIDR_A"

gcloud compute networks subnets create "$SUBNET_B" \
  --network="$VPC_B" \
  --region="$REGION" \
  --range="$CIDR_B"
```

## 3. Create HA VPN Gateways

Create one HA VPN gateway in each VPC.

```shell
gcloud compute vpn-gateways create "$GW_A" \
  --network="$VPC_A" \
  --region="$REGION" \
  --stack-type=IPV4_ONLY

gcloud compute vpn-gateways create "$GW_B" \
  --network="$VPC_B" \
  --region="$REGION" \
  --stack-type=IPV4_ONLY
```

Check the gateway interfaces and public IPs:

```shell
gcloud compute vpn-gateways describe "$GW_A" \
  --region="$REGION" \
  --format="table(name,vpnInterfaces[].id,vpnInterfaces[].ipAddress)"

gcloud compute vpn-gateways describe "$GW_B" \
  --region="$REGION" \
  --format="table(name,vpnInterfaces[].id,vpnInterfaces[].ipAddress)"
```

## 4. Create Cloud Routers

Cloud Router exchanges routes dynamically over BGP. Use a different private ASN for each VPC.

```shell
gcloud compute routers create "$ROUTER_A" \
  --region="$REGION" \
  --network="$VPC_A" \
  --asn="$ASN_A"

gcloud compute routers create "$ROUTER_B" \
  --region="$REGION" \
  --network="$VPC_B" \
  --asn="$ASN_B"
```

## 5. Create Four HA VPN Tunnels

Create two tunnels from gateway A to gateway B:

```shell
gcloud compute vpn-tunnels create "$TUNNEL_A_TO_B_0" \
  --peer-gcp-gateway="$GW_B" \
  --region="$REGION" \
  --ike-version=2 \
  --shared-secret="$SHARED_SECRET" \
  --router="$ROUTER_A" \
  --vpn-gateway="$GW_A" \
  --interface=0

gcloud compute vpn-tunnels create "$TUNNEL_A_TO_B_1" \
  --peer-gcp-gateway="$GW_B" \
  --region="$REGION" \
  --ike-version=2 \
  --shared-secret="$SHARED_SECRET" \
  --router="$ROUTER_A" \
  --vpn-gateway="$GW_A" \
  --interface=1
```

Create two matching tunnels from gateway B to gateway A:

```shell
gcloud compute vpn-tunnels create "$TUNNEL_B_TO_A_0" \
  --peer-gcp-gateway="$GW_A" \
  --region="$REGION" \
  --ike-version=2 \
  --shared-secret="$SHARED_SECRET" \
  --router="$ROUTER_B" \
  --vpn-gateway="$GW_B" \
  --interface=0

gcloud compute vpn-tunnels create "$TUNNEL_B_TO_A_1" \
  --peer-gcp-gateway="$GW_A" \
  --region="$REGION" \
  --ike-version=2 \
  --shared-secret="$SHARED_SECRET" \
  --router="$ROUTER_B" \
  --vpn-gateway="$GW_B" \
  --interface=1
```

## 6. Configure BGP Interfaces And Peers

BGP uses link-local `169.254.0.0/16` addresses. Each tunnel needs a unique `/30`.

| Tunnel      |      Router A IP |      Router B IP | Peer ASN from A | Peer ASN from B |
| ----------- | ---------------: | ---------------: | --------------: | --------------: |
| interface 0 | `169.254.0.1/30` | `169.254.0.2/30` |         `65002` |         `65001` |
| interface 1 | `169.254.1.1/30` | `169.254.1.2/30` |         `65002` |         `65001` |
|             |                  |                  |                 |                 |

Configure Cloud Router A:

```shell
gcloud compute routers add-interface "$ROUTER_A" \
  --interface-name="if-a-to-b-0" \
  --ip-address="169.254.0.1" \
  --mask-length=30 \
  --vpn-tunnel="$TUNNEL_A_TO_B_0" \
  --region="$REGION"

gcloud compute routers add-bgp-peer "$ROUTER_A" \
  --peer-name="peer-a-to-b-0" \
  --interface="if-a-to-b-0" \
  --peer-ip-address="169.254.0.2" \
  --peer-asn="$ASN_B" \
  --region="$REGION"

gcloud compute routers add-interface "$ROUTER_A" \
  --interface-name="if-a-to-b-1" \
  --ip-address="169.254.1.1" \
  --mask-length=30 \
  --vpn-tunnel="$TUNNEL_A_TO_B_1" \
  --region="$REGION"

gcloud compute routers add-bgp-peer "$ROUTER_A" \
  --peer-name="peer-a-to-b-1" \
  --interface="if-a-to-b-1" \
  --peer-ip-address="169.254.1.2" \
  --peer-asn="$ASN_B" \
  --region="$REGION"
```

Configure Cloud Router B:

```shell
gcloud compute routers add-interface "$ROUTER_B" \
  --interface-name="if-b-to-a-0" \
  --ip-address="169.254.0.2" \
  --mask-length=30 \
  --vpn-tunnel="$TUNNEL_B_TO_A_0" \
  --region="$REGION"

gcloud compute routers add-bgp-peer "$ROUTER_B" \
  --peer-name="peer-b-to-a-0" \
  --interface="if-b-to-a-0" \
  --peer-ip-address="169.254.0.1" \
  --peer-asn="$ASN_A" \
  --region="$REGION"

gcloud compute routers add-interface "$ROUTER_B" \
  --interface-name="if-b-to-a-1" \
  --ip-address="169.254.1.2" \
  --mask-length=30 \
  --vpn-tunnel="$TUNNEL_B_TO_A_1" \
  --region="$REGION"

gcloud compute routers add-bgp-peer "$ROUTER_B" \
  --peer-name="peer-b-to-a-1" \
  --interface="if-b-to-a-1" \
  --peer-ip-address="169.254.1.1" \
  --peer-asn="$ASN_A" \
  --region="$REGION"
```

## 7. Verify VPN And BGP Status

Wait a few minutes after creating the tunnels and BGP peers.

Check VPN tunnel status:

```shell
gcloud compute vpn-tunnels list \
  --regions="$REGION" \
  --format="table(name,region,vpnGateway.basename(),vpnGatewayInterface,peerGcpGateway.basename(),status)"
```

Expected result:

```text
status = ESTABLISHED
```

Check BGP status:

```shell
gcloud compute routers get-status "$ROUTER_A" \
  --region="$REGION" \
  --format="table(result.bgpPeerStatus[].name,result.bgpPeerStatus[].status,result.bgpPeerStatus[].ipAddress,result.bgpPeerStatus[].peerIpAddress)"

gcloud compute routers get-status "$ROUTER_B" \
  --region="$REGION" \
  --format="table(result.bgpPeerStatus[].name,result.bgpPeerStatus[].status,result.bgpPeerStatus[].ipAddress,result.bgpPeerStatus[].peerIpAddress)"
```

Expected result:

```text
status = ESTABLISHED
```

Check learned routes:

```shell
gcloud compute routes list \
  --filter="network:($VPC_A OR $VPC_B)" \
  --format="table(name,network.basename(),destRange,nextHopVpnTunnel.basename(),priority)"
```

## 8. Create Test VMs

Create one VM in each VPC.

```shell
gcloud compute instances create vm-a \
  --zone="$ZONE" \
  --machine-type=e2-micro \
  --network="$VPC_A" \
  --subnet="$SUBNET_A" \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --tags=ha-vpn-test

gcloud compute instances create vm-b \
  --zone="$ZONE" \
  --machine-type=e2-micro \
  --network="$VPC_B" \
  --subnet="$SUBNET_B" \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --tags=ha-vpn-test
```

Allow ICMP and SSH from the opposite VPC CIDR:

```shell
gcloud compute firewall-rules create allow-vpc-a-from-vpc-b \
  --network="$VPC_A" \
  --direction=INGRESS \
  --action=ALLOW \
  --source-ranges="$CIDR_B" \
  --target-tags=ha-vpn-test \
  --rules=icmp,tcp:22

gcloud compute firewall-rules create allow-vpc-b-from-vpc-a \
  --network="$VPC_B" \
  --direction=INGRESS \
  --action=ALLOW \
  --source-ranges="$CIDR_A" \
  --target-tags=ha-vpn-test \
  --rules=icmp,tcp:22
```

Get the private IPs:

```shell
gcloud compute instances list \
  --filter="name:(vm-a OR vm-b)" \
  --format="table(name,zone.basename(),networkInterfaces[0].networkIP,networkInterfaces[0].network.basename())"
```

Test from `vm-a` to the private IP of `vm-b`:

```shell
export VM_B_PRIVATE_IP="$(gcloud compute instances describe vm-b --zone="$ZONE" --format='value(networkInterfaces[0].networkIP)')"

gcloud compute ssh vm-a \
  --zone="$ZONE" \
  --command="ping -c 4 $VM_B_PRIVATE_IP"
```

Test from `vm-b` to the private IP of `vm-a`:

```shell
export VM_A_PRIVATE_IP="$(gcloud compute instances describe vm-a --zone="$ZONE" --format='value(networkInterfaces[0].networkIP)')"

gcloud compute ssh vm-b \
  --zone="$ZONE" \
  --command="ping -c 4 $VM_A_PRIVATE_IP"
```

## 9. Troubleshooting Commands

Describe VPN tunnels:

```shell
gcloud compute vpn-tunnels describe "$TUNNEL_A_TO_B_0" --region="$REGION"
gcloud compute vpn-tunnels describe "$TUNNEL_A_TO_B_1" --region="$REGION"
gcloud compute vpn-tunnels describe "$TUNNEL_B_TO_A_0" --region="$REGION"
gcloud compute vpn-tunnels describe "$TUNNEL_B_TO_A_1" --region="$REGION"
```

Describe routers:

```shell
gcloud compute routers describe "$ROUTER_A" --region="$REGION"
gcloud compute routers describe "$ROUTER_B" --region="$REGION"
```

Check the firewall rules that allow test traffic:

```shell
gcloud compute firewall-rules list \
  --filter="name:(allow-vpc-a-from-vpc-b OR allow-vpc-b-from-vpc-a)" \
  --format="table(name,network.basename(),direction,sourceRanges,allowed)"
```

Common issues:

- Tunnel is not `ESTABLISHED`: confirm both matching tunnels exist and use the same pre-shared key.
- BGP is not `ESTABLISHED`: confirm peer IPs are reversed correctly on the other router.
- Ping fails but BGP is up: confirm firewall rules allow ICMP from the remote VPC CIDR.
- Routes missing: confirm the VPC subnet ranges do not overlap and Cloud Router BGP sessions are established.

## 10. Cleanup

Delete test VMs and firewall rules:

```shell
gcloud compute instances delete vm-a --zone="$ZONE" --quiet
gcloud compute instances delete vm-b --zone="$ZONE" --quiet

gcloud compute firewall-rules delete allow-vpc-a-from-vpc-b --quiet
gcloud compute firewall-rules delete allow-vpc-b-from-vpc-a --quiet
```

Delete BGP peers and router interfaces:

```shell
gcloud compute routers remove-bgp-peer "$ROUTER_A" --peer-name="peer-a-to-b-0" --region="$REGION"
gcloud compute routers remove-bgp-peer "$ROUTER_A" --peer-name="peer-a-to-b-1" --region="$REGION"
gcloud compute routers remove-interface "$ROUTER_A" --interface-name="if-a-to-b-0" --region="$REGION"
gcloud compute routers remove-interface "$ROUTER_A" --interface-name="if-a-to-b-1" --region="$REGION"

gcloud compute routers remove-bgp-peer "$ROUTER_B" --peer-name="peer-b-to-a-0" --region="$REGION"
gcloud compute routers remove-bgp-peer "$ROUTER_B" --peer-name="peer-b-to-a-1" --region="$REGION"
gcloud compute routers remove-interface "$ROUTER_B" --interface-name="if-b-to-a-0" --region="$REGION"
gcloud compute routers remove-interface "$ROUTER_B" --interface-name="if-b-to-a-1" --region="$REGION"
```

Delete VPN tunnels:

```shell
gcloud compute vpn-tunnels delete "$TUNNEL_A_TO_B_0" --region="$REGION" --quiet
gcloud compute vpn-tunnels delete "$TUNNEL_A_TO_B_1" --region="$REGION" --quiet
gcloud compute vpn-tunnels delete "$TUNNEL_B_TO_A_0" --region="$REGION" --quiet
gcloud compute vpn-tunnels delete "$TUNNEL_B_TO_A_1" --region="$REGION" --quiet
```

Delete HA VPN gateways and routers:

```shell
gcloud compute vpn-gateways delete "$GW_A" --region="$REGION" --quiet
gcloud compute vpn-gateways delete "$GW_B" --region="$REGION" --quiet

gcloud compute routers delete "$ROUTER_A" --region="$REGION" --quiet
gcloud compute routers delete "$ROUTER_B" --region="$REGION" --quiet
```

Delete subnets and VPC networks:

```shell
gcloud compute networks subnets delete "$SUBNET_A" --region="$REGION" --quiet
gcloud compute networks subnets delete "$SUBNET_B" --region="$REGION" --quiet

gcloud compute networks delete "$VPC_A" --quiet
gcloud compute networks delete "$VPC_B" --quiet
```
