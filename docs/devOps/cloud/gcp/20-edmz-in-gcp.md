---
tags:
  - GCP
  - EDMZ
  - load-balancer
  - GCE
  - nginx
  - hybrid-connectivity
---
# EDMZ Pattern In Google Cloud: External Load Balancer To Nginx Proxy To On-Prem

## Overview

This design builds an **external DMZ (EDMZ)** entry point in Google Cloud:

1. External users connect to a **global external Application Load Balancer**.
2. The load balancer terminates the public connection and forwards traffic to private **Compute Engine** instances.
3. The Compute Engine instances run **Nginx** as a reverse proxy.
4. Nginx creates a new outbound connection to the internal on-prem network over **HA VPN** or **Cloud Interconnect**.

This is a proxy pattern, not packet forwarding. The on-prem system sees the source as the Nginx VM private IP, while the original client IP is passed in headers such as `X-Forwarded-For`.

> Official references:
> - [Global external Application Load Balancer overview](https://cloud.google.com/load-balancing/docs/https)
> - [Backend service overview](https://cloud.google.com/load-balancing/docs/backend-service)
> - [Google Cloud Armor overview](https://cloud.google.com/armor/docs/security-policy-overview)
> - [HA VPN overview](https://cloud.google.com/network-connectivity/docs/vpn/concepts/overview)
> - [Cloud Interconnect overview](https://cloud.google.com/network-connectivity/docs/interconnect/concepts/overview)

## Target Architecture

```text
Internet users
    |
    | HTTPS 443
    v
Global external Application Load Balancer
    |
    | HTTP 80 or HTTPS to backend
    v
Managed Instance Group: edmz-nginx
    |
    | New private connection from Nginx VM IP
    v
HA VPN / Cloud Interconnect
    |
    v
On-prem internal application
```

Recommended production components:

- **Global external Application Load Balancer** for public ingress.
- **Google Cloud Armor** for WAF, IP allow/deny lists, and DDoS-aware edge policy.
- **Managed instance group (MIG)** for Nginx proxy VMs.
- **Private-only VM NICs**. Do not assign external IPs to the Nginx instances.
- **Cloud NAT** only if the Nginx VMs need outbound internet for package updates.
- **HA VPN** for encrypted site-to-site connectivity, or **Dedicated/Partner Interconnect** for high-throughput private connectivity.
- **Cloud Router** with BGP for dynamic route exchange to on-prem.
- **Firewall rules** that allow only load-balancer health-check/proxy ranges into Nginx and only required destination ports from Nginx to on-prem.

## Important Design Decisions

### Use Nginx As A Reverse Proxy

Nginx should terminate the load-balancer backend request and open a new connection to the on-prem application:

```nginx
location / {
    proxy_pass http://ONPREM_APP_IP_OR_DNS:ONPREM_APP_PORT;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

This gives you a clean security boundary:

- Public clients never route directly to on-prem.
- On-prem firewall rules only need to trust the GCP EDMZ subnet or Nginx VM IP range.
- You can inspect, log, restrict, and transform requests at Nginx.

### Do Not Use This As Transparent NAT

This pattern does not preserve the original client source IP at the TCP/IP layer. The on-prem system receives a connection from the Nginx VM private IP.

If the on-prem application needs the real client IP, use application headers:

- `X-Forwarded-For`
- `X-Real-IP`
- `Forwarded`

The on-prem application must trust these headers only from the Nginx proxy source range.

### Prefer HTTPS At The Edge

Use HTTPS from the external user to the Google Cloud load balancer. For the load balancer to Nginx hop, choose one:

- HTTP backend: simpler, acceptable only inside a tightly controlled VPC.
- HTTPS backend: stronger, recommended when compliance requires encryption on every hop.

## Prerequisites

Run commands from **Google Cloud Shell** or a workstation with Google Cloud CLI installed and authenticated.

Required IAM permissions are typically covered by these roles for a lab:

- `roles/compute.networkAdmin`
- `roles/compute.instanceAdmin.v1`
- `roles/iam.serviceAccountUser`
- `roles/serviceusage.serviceUsageAdmin`

Enable APIs:

```shell
gcloud services enable compute.googleapis.com
```

Set the project:

```shell
gcloud auth login
gcloud config set project PROJECT_ID
gcloud config list --format='text(core.project)'
```

## Variables

Adjust these values for your environment.

```shell
export PROJECT_ID="$(gcloud config get-value project)"
export REGION="us-central1"
export ZONE="us-central1-a"

export VPC="edmz-vpc"
export EDMZ_SUBNET="edmz-subnet"
export EDMZ_CIDR="10.10.10.0/24"

export ONPREM_CIDR="172.16.0.0/16"
export ONPREM_APP_HOST="172.16.10.20"
export ONPREM_APP_PORT="8080"

export TEMPLATE="edmz-nginx-template"
export MIG="edmz-nginx-mig"
export BACKEND_TAG="edmz-nginx"

export HEALTH_CHECK="edmz-nginx-health"
export BACKEND_SERVICE="edmz-nginx-backend"
export URL_MAP="edmz-url-map"
export HTTPS_PROXY="edmz-https-proxy"
export HTTP_PROXY="edmz-http-proxy"
export FORWARDING_RULE_HTTPS="edmz-https-rule"
export FORWARDING_RULE_HTTP="edmz-http-rule"
export LB_IP_NAME="edmz-lb-ip"

export DOMAIN_NAME="edmz.example.com"
export SSL_CERT="edmz-managed-cert"

gcloud config set compute/region "$REGION"
gcloud config set compute/zone "$ZONE"
```

## 1. Create The EDMZ VPC And Subnet

Use a custom VPC so the EDMZ IP range is explicit and does not overlap with on-prem.

```shell
gcloud compute networks create "$VPC" \
  --subnet-mode=custom \
  --bgp-routing-mode=global

gcloud compute networks subnets create "$EDMZ_SUBNET" \
  --network="$VPC" \
  --region="$REGION" \
  --range="$EDMZ_CIDR"
```

## 2. Connect GCP To On-Prem

Create one of these hybrid connectivity options:

- **HA VPN**: encrypted IPsec tunnel over the internet. Good default for many EDMZ deployments.
- **Cloud Interconnect**: private physical or partner connectivity. Use this for higher throughput, lower latency, or private connectivity requirements.

For an HA VPN lab, use the existing guide in this repo:

- [GCP HA VPN Lab: Connect Two VPC Networks](./19-HA-VPN.md)

For a real on-prem connection:

1. Create an HA VPN gateway in the EDMZ VPC.
2. Create a Cloud Router in the same region.
3. Create redundant VPN tunnels.
4. Configure BGP between Cloud Router and the on-prem router.
5. Advertise the EDMZ subnet to on-prem.
6. Learn on-prem application routes in Google Cloud.

Validate that routes to on-prem exist:

```shell
gcloud compute routes list \
  --filter="network:$VPC AND destRange:$ONPREM_CIDR"
```

Validate private connectivity from a temporary VM in the EDMZ subnet before adding the load balancer:

```shell
curl -v "http://${ONPREM_APP_HOST}:${ONPREM_APP_PORT}/"
```

## 3. Create Nginx Startup Script

This startup script installs Nginx and configures it as a reverse proxy to the on-prem application.

```shell
cat > nginx-startup.sh <<'EOF'
#!/bin/bash
set -euo pipefail

apt-get update
apt-get install -y nginx

ONPREM_APP_HOST="$(curl -fsH 'Metadata-Flavor: Google' \
  http://metadata.google.internal/computeMetadata/v1/instance/attributes/ONPREM_APP_HOST)"
ONPREM_APP_PORT="$(curl -fsH 'Metadata-Flavor: Google' \
  http://metadata.google.internal/computeMetadata/v1/instance/attributes/ONPREM_APP_PORT)"

cat > /etc/nginx/conf.d/edmz.conf <<'NGINX'
server {
    listen 80 default_server;
    server_name _;

    location /healthz {
        access_log off;
        return 200 "ok\n";
        add_header Content-Type text/plain;
    }

    location / {
        proxy_pass http://ONPREM_APP_HOST:ONPREM_APP_PORT;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_connect_timeout 5s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
}
NGINX

sed -i "s/ONPREM_APP_HOST/${ONPREM_APP_HOST}/g" /etc/nginx/conf.d/edmz.conf
sed -i "s/ONPREM_APP_PORT/${ONPREM_APP_PORT}/g" /etc/nginx/conf.d/edmz.conf

rm -f /etc/nginx/sites-enabled/default
nginx -t
systemctl enable nginx
systemctl restart nginx
EOF
```

> **Note:** In production, consider using a managed image, instance template metadata, or configuration management instead of embedding application routing directly in a startup script.

## 4. Create The Nginx Managed Instance Group

Create an instance template with private-only VMs:

```shell
gcloud compute instance-templates create "$TEMPLATE" \
  --region="$REGION" \
  --network="$VPC" \
  --subnet="$EDMZ_SUBNET" \
  --tags="$BACKEND_TAG" \
  --machine-type=e2-medium \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --no-address \
  --metadata=ONPREM_APP_HOST="$ONPREM_APP_HOST",ONPREM_APP_PORT="$ONPREM_APP_PORT" \
  --metadata-from-file=startup-script=nginx-startup.sh
```

Create a regional MIG:

```shell
gcloud compute instance-groups managed create "$MIG" \
  --region="$REGION" \
  --template="$TEMPLATE" \
  --size=2
```

Set the named port used by the load balancer backend service:

```shell
gcloud compute instance-groups managed set-named-ports "$MIG" \
  --region="$REGION" \
  --named-ports=http:80
```

## 5. Create Firewall Rules

Allow load balancer proxy and health-check traffic to Nginx.

The documented Google Cloud health check and proxy source ranges for external Application Load Balancers include:

- `35.191.0.0/16`
- `130.211.0.0/22`

```shell
gcloud compute firewall-rules create allow-lb-to-edmz-nginx \
  --network="$VPC" \
  --direction=INGRESS \
  --action=ALLOW \
  --source-ranges=35.191.0.0/16,130.211.0.0/22 \
  --target-tags="$BACKEND_TAG" \
  --rules=tcp:80
```

Allow Nginx to connect to the on-prem application port:

```shell
gcloud compute firewall-rules create allow-edmz-nginx-to-onprem \
  --network="$VPC" \
  --direction=EGRESS \
  --action=ALLOW \
  --destination-ranges="$ONPREM_CIDR" \
  --target-tags="$BACKEND_TAG" \
  --rules=tcp:"$ONPREM_APP_PORT"
```

On the on-prem firewall, allow traffic from the EDMZ subnet or specific Nginx VM IPs to the internal application:

```text
source:      10.10.10.0/24
destination: 172.16.10.20
port:        tcp/8080
action:      allow
```

## 6. Create The Global External Application Load Balancer

Reserve a global IP address:

```shell
gcloud compute addresses create "$LB_IP_NAME" \
  --ip-version=IPV4 \
  --global

gcloud compute addresses describe "$LB_IP_NAME" \
  --global \
  --format='value(address)'
```

Create the health check:

```shell
gcloud compute health-checks create http "$HEALTH_CHECK" \
  --port=80 \
  --request-path=/healthz \
  --check-interval=10s \
  --timeout=5s \
  --healthy-threshold=2 \
  --unhealthy-threshold=2
```

Create the backend service:

```shell
gcloud compute backend-services create "$BACKEND_SERVICE" \
  --load-balancing-scheme=EXTERNAL_MANAGED \
  --protocol=HTTP \
  --port-name=http \
  --health-checks="$HEALTH_CHECK" \
  --global
```

Add the MIG as the backend:

```shell
gcloud compute backend-services add-backend "$BACKEND_SERVICE" \
  --instance-group="$MIG" \
  --instance-group-region="$REGION" \
  --global
```

Create the URL map:

```shell
gcloud compute url-maps create "$URL_MAP" \
  --default-service="$BACKEND_SERVICE"
```

Create a Google-managed SSL certificate:

```shell
gcloud compute ssl-certificates create "$SSL_CERT" \
  --domains="$DOMAIN_NAME" \
  --global
```

Create the HTTPS proxy:

```shell
gcloud compute target-https-proxies create "$HTTPS_PROXY" \
  --url-map="$URL_MAP" \
  --ssl-certificates="$SSL_CERT"
```

Create the HTTPS forwarding rule:

```shell
gcloud compute forwarding-rules create "$FORWARDING_RULE_HTTPS" \
  --load-balancing-scheme=EXTERNAL_MANAGED \
  --network-tier=PREMIUM \
  --address="$LB_IP_NAME" \
  --global \
  --target-https-proxy="$HTTPS_PROXY" \
  --ports=443
```

Optional HTTP listener:

```shell
gcloud compute target-http-proxies create "$HTTP_PROXY" \
  --url-map="$URL_MAP"

gcloud compute forwarding-rules create "$FORWARDING_RULE_HTTP" \
  --load-balancing-scheme=EXTERNAL_MANAGED \
  --network-tier=PREMIUM \
  --address="$LB_IP_NAME" \
  --global \
  --target-http-proxy="$HTTP_PROXY" \
  --ports=80
```

## 7. Add Google Cloud Armor

Attach a Cloud Armor policy to restrict and inspect public traffic before it reaches the EDMZ proxy.

Example policy:

Cloud Armor creates a default rule for each security policy. Its reserved priority is `2147483647` (`INT-MAX`), the lowest priority, so it applies only when no higher-priority rule matches.

```shell
gcloud compute security-policies create edmz-edge-policy \
  --description="Edge policy for EDMZ ingress"

gcloud compute security-policies rules create 1000 \
  --security-policy=edmz-edge-policy \
  --expression="origin.ip in ['203.0.113.0/24']" \
  --action=allow \
  --description="Allow trusted client range"

gcloud compute security-policies rules update 2147483647 \
  --security-policy=edmz-edge-policy \
  --action=deny-403

gcloud compute backend-services update "$BACKEND_SERVICE" \
  --security-policy=edmz-edge-policy \
  --global
```

> **Warning:** Replace `203.0.113.0/24` with your real trusted source range. If this is a public application, use Cloud Armor managed WAF rules instead of a strict IP allowlist.

## 8. DNS

Create or update the public DNS record for the application:

```text
edmz.example.com.  A  LOAD_BALANCER_GLOBAL_IP
```

After DNS points to the load balancer IP, Google-managed certificate provisioning can complete.

Check the certificate:

```shell
gcloud compute ssl-certificates describe "$SSL_CERT" \
  --global \
  --format='get(managed.status,managed.domainStatus)'
```

## Validation

Check backend health:

```shell
gcloud compute backend-services get-health "$BACKEND_SERVICE" \
  --global
```

Check that Nginx VMs have no external IP:

```shell
gcloud compute instances list \
  --filter="name~'${MIG}'" \
  --format="table(name,zone,networkInterfaces[0].networkIP,networkInterfaces[0].accessConfigs[0].natIP)"
```

Test the public endpoint:

```shell
curl -v "https://${DOMAIN_NAME}/"
```

Check Nginx logs on a backend VM:

```shell
gcloud compute ssh INSTANCE_NAME \
  --zone=ZONE \
  --tunnel-through-iap \
  --command="sudo tail -n 50 /var/log/nginx/access.log"
```

Validate on-prem sees only EDMZ proxy source IPs:

```text
Expected on-prem source IP: Nginx VM private IP from 10.10.10.0/24
Expected original client IP: X-Forwarded-For header
```

## Troubleshooting

### Backend Is Unhealthy

Check:

- Firewall allows `35.191.0.0/16` and `130.211.0.0/22` to TCP port `80`.
- Nginx is listening on port `80`.
- `/healthz` returns HTTP `200`.
- The MIG named port is `http:80`.

Commands:

```shell
gcloud compute backend-services get-health "$BACKEND_SERVICE" --global
gcloud compute firewall-rules describe allow-lb-to-edmz-nginx
```

### Nginx Cannot Reach On-Prem

Check:

- GCP route to the on-prem CIDR exists.
- On-prem route back to `EDMZ_CIDR` exists.
- On-prem firewall allows the Nginx VM source range.
- The on-prem application is listening on the expected port.

Commands:

```shell
gcloud compute routes list \
  --filter="network:$VPC AND destRange:$ONPREM_CIDR"

gcloud compute instances list \
  --filter="tags.items=$BACKEND_TAG" \
  --format="table(name,zone,networkInterfaces[0].networkIP)"
```

### Client IP Is Missing On-Prem

This is expected at the network layer. Nginx creates a new TCP connection to on-prem.

Use `X-Forwarded-For` or `X-Real-IP` at the application layer, and configure the on-prem application to trust those headers only from the Nginx proxy IP range.

## Security Notes

- Do not assign external IP addresses to the Nginx VMs.
- Do not allow direct internet ingress to the EDMZ subnet.
- Restrict inbound GCP firewall rules to load-balancer proxy and health-check ranges only.
- Restrict outbound firewall rules from Nginx to only required on-prem CIDRs and ports.
- Use Cloud Armor for edge allowlists, rate limiting, and WAF rules.
- Use IAP or a bastion pattern for administrator SSH access.
- Enable load balancer logging and Nginx access logs for audit and troubleshooting.
- Consider Private Service Connect, hybrid NEGs, or Apigee if the use case is API publishing rather than generic reverse proxying.

## Cleanup

> **Warning:** These commands delete the example load balancer, backend service, MIG, firewall rules, and network resources. Run only in a lab project or after confirming the resources are no longer needed.

```shell
gcloud compute forwarding-rules delete "$FORWARDING_RULE_HTTPS" --global --quiet
gcloud compute forwarding-rules delete "$FORWARDING_RULE_HTTP" --global --quiet
gcloud compute target-https-proxies delete "$HTTPS_PROXY" --quiet
gcloud compute target-http-proxies delete "$HTTP_PROXY" --quiet
gcloud compute ssl-certificates delete "$SSL_CERT" --global --quiet
gcloud compute url-maps delete "$URL_MAP" --quiet
gcloud compute backend-services delete "$BACKEND_SERVICE" --global --quiet
gcloud compute health-checks delete "$HEALTH_CHECK" --quiet
gcloud compute instance-groups managed delete "$MIG" --region="$REGION" --quiet
gcloud compute instance-templates delete "$TEMPLATE" --quiet
gcloud compute firewall-rules delete allow-lb-to-edmz-nginx --quiet
gcloud compute firewall-rules delete allow-edmz-nginx-to-onprem --quiet
gcloud compute addresses delete "$LB_IP_NAME" --global --quiet
gcloud compute networks subnets delete "$EDMZ_SUBNET" --region="$REGION" --quiet
gcloud compute networks delete "$VPC" --quiet
```

Delete HA VPN, Cloud Router, or Interconnect resources separately if they were created for this environment.

## References

- [Global external Application Load Balancer overview](https://cloud.google.com/load-balancing/docs/https)
- [Backend services](https://cloud.google.com/load-balancing/docs/backend-service)
- [Health checks overview](https://cloud.google.com/load-balancing/docs/health-check-concepts)
- [Firewall rules for load balancers](https://cloud.google.com/load-balancing/docs/firewall-rules)
- [Google Cloud Armor security policy overview](https://cloud.google.com/armor/docs/security-policy-overview)
- [HA VPN overview](https://cloud.google.com/network-connectivity/docs/vpn/concepts/overview)
- [Cloud Router overview](https://cloud.google.com/network-connectivity/docs/router/concepts/overview)
- [Cloud Interconnect overview](https://cloud.google.com/network-connectivity/docs/interconnect/concepts/overview)
- [gcloud compute reference](https://cloud.google.com/sdk/gcloud/reference/compute)
