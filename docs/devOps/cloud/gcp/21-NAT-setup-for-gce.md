---
tags:
  - GCP
  - Cloud-NAT
  - GCE
  - networking
  - internet-egress
---
# Cloud NAT Setup For GCE Without External IP

## Overview

Use **Cloud NAT** when existing Compute Engine VM instances have only internal IP addresses but need outbound internet access for tasks such as:

- OS package updates
- Downloading dependencies
- Calling external APIs
- Pulling artifacts from public repositories

Cloud NAT provides outbound source NAT for matching private resources. It does **not** allow unsolicited inbound internet connections to the VMs.

```text
Private GCE VM
  internal IP only
      |
      | outbound connection
      v
Cloud NAT gateway
      |
      | translated source IP
      v
Internet
```

> Official references:
> - [Cloud NAT overview](https://cloud.google.com/nat/docs/overview)
> - [Set up and manage Cloud NAT](https://cloud.google.com/nat/docs/set-up-manage-network-address-translation)
> - [gcloud compute routers create](https://cloud.google.com/sdk/gcloud/reference/compute/routers/create)
> - [gcloud compute routers nats create](https://cloud.google.com/sdk/gcloud/reference/compute/routers/nats/create)
> - [VPC routes](https://cloud.google.com/vpc/docs/routes)

## Important Behavior

- Cloud NAT is regional. Create it in the same region as the subnet used by the private GCE instances.
- Cloud NAT is configured on a **Cloud Router**, but it does not require BGP unless the router is also used for VPN or Interconnect.
- Cloud NAT is distributed and managed by Google Cloud; it is not a NAT VM.
- Existing VMs do not need to be recreated. NAT applies after the subnet or IP range is included in the Cloud NAT configuration.
- Public Cloud NAT requires a route to `0.0.0.0/0` whose next hop is the default internet gateway.
- Egress firewall rules still apply. If your VPC blocks outbound traffic, allow the required destination ports.
- For access to Google APIs from private VMs, also consider **Private Google Access**. Cloud NAT is mainly for public internet destinations.

## Prerequisites

Run these commands from **Google Cloud Shell** or a workstation with Google Cloud CLI installed and authenticated.

Required IAM permissions are typically covered by:

- `roles/compute.networkAdmin`
- `roles/serviceusage.serviceUsageAdmin`

Enable the Compute Engine API:

```shell
gcloud services enable compute.googleapis.com
```

Set your project:

```shell
gcloud auth login
gcloud config set project PROJECT_ID
gcloud config list --format='text(core.project)'
```

## Variables

Update these values to match the existing GCE environment.

```shell
export PROJECT_ID="$(gcloud config get-value project)"
export REGION="us-central1"
export ZONE="us-central1-a"

export VPC="default"
export SUBNET="default"

export ROUTER="nat-router-${REGION}"
export NAT_NAME="nat-gateway-${REGION}"
export NAT_IP_NAME="nat-ip-${REGION}"

gcloud config set compute/region "$REGION"
gcloud config set compute/zone "$ZONE"
```

Find the network and subnet for an existing private VM:

```shell
gcloud compute instances describe INSTANCE_NAME \
  --zone=ZONE \
  --format="table(networkInterfaces[0].network.basename(),networkInterfaces[0].subnetwork.basename(),networkInterfaces[0].networkIP,networkInterfaces[0].accessConfigs[0].natIP)"
```

If `natIP` is empty, the VM does not have an external IP address.

## 1. Verify The Subnet And Default Route

List the subnet:

```shell
gcloud compute networks subnets describe "$SUBNET" \
  --region="$REGION" \
  --format="yaml(name,network,ipCidrRange,privateIpGoogleAccess)"
```

Verify that the VPC has a default route to the default internet gateway:

```shell
gcloud compute routes list \
  --filter="network:$VPC AND destRange=0.0.0.0/0" \
  --format="table(name,destRange,nextHopGateway,priority)"
```

Expected `nextHopGateway`:

```text
default-internet-gateway
```

If the default route was deleted intentionally, confirm the network design before adding it back.

```shell
gcloud compute routes create default-route-to-internet \
  --network="$VPC" \
  --destination-range=0.0.0.0/0 \
  --next-hop-gateway=default-internet-gateway
```

## 2. Create A Cloud Router

Cloud NAT requires a Cloud Router in the same region as the NAT gateway.

```shell
gcloud compute routers create "$ROUTER" \
  --network="$VPC" \
  --region="$REGION"
```

Validate:

```shell
gcloud compute routers describe "$ROUTER" \
  --region="$REGION" \
  --format="yaml(name,network,region)"
```

## 3. Option A: Create Cloud NAT With Auto-Allocated External IPs

This is the simplest setup. Google Cloud automatically allocates regional external IP addresses for NAT.

Use this when:

- You do not need a fixed outbound source IP.
- External services do not require IP allowlisting.
- You want the fastest working setup.

```shell
gcloud compute routers nats create "$NAT_NAME" \
  --router="$ROUTER" \
  --region="$REGION" \
  --type=PUBLIC \
  --auto-allocate-nat-external-ips \
  --nat-custom-subnet-ip-ranges="$SUBNET:ALL" \
  --enable-logging \
  --log-filter=ERRORS_ONLY
```

## 4. Option B: Create Cloud NAT With A Static External IP

Use this when external partners, firewalls, or APIs require a stable outbound source IP.

Reserve a regional external IP address:

```shell
gcloud compute addresses create "$NAT_IP_NAME" \
  --region="$REGION" \
  --network-tier=PREMIUM
```

Check the reserved IP:

```shell
gcloud compute addresses describe "$NAT_IP_NAME" \
  --region="$REGION" \
  --format="value(address)"
```

Create the NAT gateway using the reserved IP:

```shell
gcloud compute routers nats create "$NAT_NAME" \
  --router="$ROUTER" \
  --region="$REGION" \
  --type=PUBLIC \
  --nat-external-ip-pool="$NAT_IP_NAME" \
  --nat-custom-subnet-ip-ranges="$SUBNET:ALL" \
  --enable-logging \
  --log-filter=ERRORS_ONLY
```

> **Note:** Use either Option A or Option B, not both with the same `NAT_NAME`.

## 5. NAT Scope Choices

The examples above apply NAT to all primary and secondary IP ranges in one subnet:

```shell
--nat-custom-subnet-ip-ranges="$SUBNET:ALL"
```

Other common choices:

Apply NAT to all subnets in the region:

```shell
--nat-all-subnet-ip-ranges
```

Apply NAT only to primary IP ranges of all regional subnets:

```shell
--nat-primary-subnet-ip-ranges
```

For production, prefer the smallest scope that matches the workloads that need outbound internet access.

## 6. Egress Firewall Rules

Cloud NAT does not bypass VPC firewall rules. If your VPC has custom deny rules or a restrictive firewall policy, allow the required outbound traffic.

Example: allow private VMs in the subnet to use HTTP and HTTPS egress.

```shell
gcloud compute firewall-rules create allow-private-vms-web-egress \
  --network="$VPC" \
  --direction=EGRESS \
  --action=ALLOW \
  --destination-ranges=0.0.0.0/0 \
  --rules=tcp:80,tcp:443
```

> **Security note:** Avoid broad egress if your environment requires strict outbound control. Prefer destination allowlists, firewall policies, or proxy-based egress for regulated workloads.

## 7. Validate From An Existing Private VM

SSH to the VM. If the VM has no external IP, use IAP TCP forwarding or a bastion host.

```shell
gcloud compute ssh INSTANCE_NAME \
  --zone=ZONE \
  --tunnel-through-iap
```

From inside the VM, test external internet access:

```shell
curl -I https://www.google.com
```

Check the public egress IP:

```shell
curl -s https://ifconfig.me
```

If you used a static NAT IP, the output should match:

```shell
gcloud compute addresses describe "$NAT_IP_NAME" \
  --region="$REGION" \
  --format="value(address)"
```

Test package update access:

```shell
sudo apt-get update
```

## 8. Validate The NAT Configuration

Describe the NAT:

```shell
gcloud compute routers nats describe "$NAT_NAME" \
  --router="$ROUTER" \
  --region="$REGION" \
  --format="yaml"
```

List NATs on the router:

```shell
gcloud compute routers nats list \
  --router="$ROUTER" \
  --region="$REGION"
```

Check logs in Cloud Logging if logging was enabled:

```shell
gcloud logging read \
  'resource.type="nat_gateway"' \
  --limit=20 \
  --format="table(timestamp,resource.labels.gateway_name,jsonPayload.allocation_status,jsonPayload.connection)"
```

## Troubleshooting

### VM Still Cannot Access The Internet

Check these items:

- The VM is in the same region and subnet covered by the Cloud NAT rule.
- The VM has no external IP, or outbound traffic is still expected to use Cloud NAT.
- The VPC has a default route to `default-internet-gateway`.
- Egress firewall rules allow the destination protocol and port.
- The guest OS has working DNS configuration.
- The destination service is reachable and not blocking the NAT external IP.

Commands:

```shell
gcloud compute instances describe INSTANCE_NAME \
  --zone=ZONE \
  --format="yaml(networkInterfaces)"

gcloud compute routes list \
  --filter="network:$VPC AND destRange=0.0.0.0/0"

gcloud compute routers nats describe "$NAT_NAME" \
  --router="$ROUTER" \
  --region="$REGION"
```

### DNS Fails But IP Connectivity Works

From the VM:

```shell
curl -I https://142.250.191.4
nslookup www.google.com
```

If IP access works but DNS fails, review the VM resolver configuration and VPC DNS policy.

### External Partner Does Not See The Expected Source IP

If you need a fixed source IP, use static NAT IPs with `--nat-external-ip-pool`. Auto-allocated NAT IPs are not appropriate for partner allowlists.

Check the current NAT IP:

```shell
curl -s https://ifconfig.me
```

### Port Exhaustion Or Intermittent Connections

High connection volume workloads can exhaust NAT source ports.

Mitigations:

- Add more NAT external IP addresses.
- Enable dynamic port allocation.
- Increase minimum ports per VM.
- Split high-volume workloads across subnets or NAT gateways.

Example update:

```shell
gcloud compute routers nats update "$NAT_NAME" \
  --router="$ROUTER" \
  --region="$REGION" \
  --enable-dynamic-port-allocation
```

## Security Notes

- Cloud NAT allows outbound internet access only; it does not expose private VMs to inbound internet connections.
- Keep VMs without external IPs unless direct inbound access is required.
- Use static NAT IPs when external destinations require source IP allowlisting.
- Use egress firewall rules to restrict destinations and ports.
- Enable NAT logging when troubleshooting or when audit requirements need it.
- Use Private Google Access for private access to Google APIs and services where appropriate.
- Monitor NAT port usage for high-connection workloads.

## Cleanup

> **Warning:** These commands remove NAT internet egress for private VMs using this NAT gateway. Run only after confirming the workloads no longer need it.

Delete the NAT configuration:

```shell
gcloud compute routers nats delete "$NAT_NAME" \
  --router="$ROUTER" \
  --region="$REGION" \
  --quiet
```

Delete the Cloud Router if it is not used for VPN, Interconnect, or another NAT:

```shell
gcloud compute routers delete "$ROUTER" \
  --region="$REGION" \
  --quiet
```

Delete the static NAT IP only if you created one and no longer need the address:

```shell
gcloud compute addresses delete "$NAT_IP_NAME" \
  --region="$REGION" \
  --quiet
```

Delete the example egress firewall rule if you created it:

```shell
gcloud compute firewall-rules delete allow-private-vms-web-egress \
  --quiet
```

## References

- [Cloud NAT overview](https://cloud.google.com/nat/docs/overview)
- [Set up and manage Cloud NAT](https://cloud.google.com/nat/docs/set-up-manage-network-address-translation)
- [Cloud NAT logging and monitoring](https://cloud.google.com/nat/docs/monitoring)
- [gcloud compute routers create](https://cloud.google.com/sdk/gcloud/reference/compute/routers/create)
- [gcloud compute routers nats create](https://cloud.google.com/sdk/gcloud/reference/compute/routers/nats/create)
- [gcloud compute addresses create](https://cloud.google.com/sdk/gcloud/reference/compute/addresses/create)
- [VPC routes](https://cloud.google.com/vpc/docs/routes)
- [Private Google Access](https://cloud.google.com/vpc/docs/private-google-access)
