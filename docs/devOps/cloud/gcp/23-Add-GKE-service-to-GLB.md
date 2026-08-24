---
tags:
  - GCP
  - GKE
  - global-load-balancer
  - standalone-neg
---

# Add a GKE Service to a Global Load Balancer

This runbook shows how to expose a GKE `Service` through a global external Application Load Balancer by using a standalone zonal network endpoint group (NEG). Use this approach when the global load balancer is managed outside GKE, or when you need to attach a GKE workload to an existing backend service, URL map, proxy, and forwarding rule.

For a new GKE-only HTTP(S) load balancer, GKE Ingress or Gateway is usually simpler because GKE manages the load balancer resources for you.

## Prerequisites

- A GKE cluster with a deployed workload.
- `gcloud` and `kubectl` installed and authenticated.
- Required permissions to administer GKE, Compute Engine load balancing resources, and firewall rules.
- The GKE Service must be reachable by Google Cloud load balancer health checks.
- Test in a non-production project before changing a production load balancer.

Enable required APIs:

```shell
gcloud services enable \
  container.googleapis.com \
  compute.googleapis.com
```

## Variables

```shell
PROJECT_ID="PROJECT_ID"
CLUSTER_NAME="CLUSTER_NAME"
REGION="REGION"
ZONE_1="ZONE_1"
ZONE_2="ZONE_2"
NAMESPACE="default"

APP_NAME="hello-app"
SERVICE_NAME="hello-service"
SERVICE_PORT="80"
TARGET_PORT="8080"

NEG_NAME="hello-service-neg"
HEALTH_CHECK_NAME="hello-http-health-check"
BACKEND_SERVICE_NAME="hello-backend-service"
URL_MAP_NAME="hello-url-map"
TARGET_PROXY_NAME="hello-http-proxy"
FORWARDING_RULE_NAME="hello-http-forwarding-rule"
GLOBAL_IP_NAME="hello-glb-ip"
```

Set the project and connect to the cluster:

```shell
gcloud config set project "${PROJECT_ID}"
gcloud container clusters get-credentials "${CLUSTER_NAME}" \
  --region="${REGION}" \
  --project="${PROJECT_ID}"
```

If the cluster is zonal, use `--zone="${ZONE_1}"` instead of `--region="${REGION}"`.

## Steps

### 1. Deploy the workload

Skip this step if the application already exists.

```shell
kubectl create deployment "${APP_NAME}" \
  --image=us-docker.pkg.dev/google-samples/containers/gke/hello-app:1.0 \
  --namespace="${NAMESPACE}"

kubectl set env deployment/"${APP_NAME}" \
  PORT="${TARGET_PORT}" \
  --namespace="${NAMESPACE}"
```

### 2. Create a Service with a standalone NEG

The annotation tells GKE to create a zonal NEG for the Service port. The NEG contains Pod IP endpoints and can be attached to a Compute Engine backend service.

```shell
kubectl apply -n "${NAMESPACE}" -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: ${SERVICE_NAME}
  annotations:
    cloud.google.com/neg: '{"exposed_ports":{"${SERVICE_PORT}":{"name":"${NEG_NAME}"}}}'
spec:
  selector:
    app: ${APP_NAME}
  type: ClusterIP
  ports:
  - name: http
    port: ${SERVICE_PORT}
    targetPort: ${TARGET_PORT}
EOF
```

Check that GKE created the NEG:

```shell
kubectl get service "${SERVICE_NAME}" \
  --namespace="${NAMESPACE}" \
  -o jsonpath='{.metadata.annotations.cloud\.google\.com/neg-status}'
```

Sample output:

```json
{"network_endpoint_groups":{"80":"hello-service-neg"},"zones":["us-central1-a","us-central1-b"]}
```

### 3. Confirm the NEG zones

Standalone NEGs are zonal. Add every zone that contains endpoints to the backend service.

```shell
gcloud compute network-endpoint-groups list \
  --filter="name=${NEG_NAME}" \
  --format="table(name,zone,networkEndpointType,size)"
```

Check endpoints in each zone:

```shell
gcloud compute network-endpoint-groups list-network-endpoints "${NEG_NAME}" \
  --zone="${ZONE_1}"
```

Repeat the endpoint check for each zone returned by the previous command.

### 4. Allow load balancer health checks

Google Cloud external Application Load Balancers use health check source ranges `130.211.0.0/22` and `35.191.0.0/16`. Allow these ranges to reach the backend port.

If your GKE nodes use network tags:

```shell
gcloud compute firewall-rules create allow-gke-neg-health-checks \
  --network=default \
  --direction=INGRESS \
  --action=ALLOW \
  --source-ranges=130.211.0.0/22,35.191.0.0/16 \
  --target-tags=GKE_NODE_NETWORK_TAG \
  --rules=tcp:${TARGET_PORT}
```

If the rule already exists, verify that it covers the correct VPC, source ranges, targets, and port:

```shell
gcloud compute firewall-rules describe allow-gke-neg-health-checks
```

### 5. Create the health check

Use a path that returns HTTP `200` when the Pod is healthy.

```shell
gcloud compute health-checks create http "${HEALTH_CHECK_NAME}" \
  --global \
  --port="${TARGET_PORT}" \
  --request-path="/" \
  --check-interval="10s" \
  --timeout="5s" \
  --healthy-threshold="2" \
  --unhealthy-threshold="2"
```

### 6. Create or update the backend service

Create the backend service if the load balancer does not already have one for this GKE Service:

```shell
gcloud compute backend-services create "${BACKEND_SERVICE_NAME}" \
  --global \
  --load-balancing-scheme=EXTERNAL_MANAGED \
  --protocol=HTTP \
  --health-checks="${HEALTH_CHECK_NAME}"
```

Attach each zonal NEG:

```shell
gcloud compute backend-services add-backend "${BACKEND_SERVICE_NAME}" \
  --global \
  --network-endpoint-group="${NEG_NAME}" \
  --network-endpoint-group-zone="${ZONE_1}" \
  --balancing-mode=RATE \
  --max-rate-per-endpoint=100
```

Repeat `add-backend` for every zone that contains the NEG, for example:

```shell
gcloud compute backend-services add-backend "${BACKEND_SERVICE_NAME}" \
  --global \
  --network-endpoint-group="${NEG_NAME}" \
  --network-endpoint-group-zone="${ZONE_2}" \
  --balancing-mode=RATE \
  --max-rate-per-endpoint=100
```

### 7. Create the global load balancer

Skip this step if you are adding the backend service to an existing URL map.

Reserve a global IP address:

```shell
gcloud compute addresses create "${GLOBAL_IP_NAME}" \
  --global \
  --ip-version=IPV4
```

Create the URL map:

```shell
gcloud compute url-maps create "${URL_MAP_NAME}" \
  --default-service="${BACKEND_SERVICE_NAME}"
```

Create the HTTP target proxy:

```shell
gcloud compute target-http-proxies create "${TARGET_PROXY_NAME}" \
  --url-map="${URL_MAP_NAME}"
```

Create the global forwarding rule:

```shell
gcloud compute forwarding-rules create "${FORWARDING_RULE_NAME}" \
  --global \
  --load-balancing-scheme=EXTERNAL_MANAGED \
  --address="${GLOBAL_IP_NAME}" \
  --target-http-proxy="${TARGET_PROXY_NAME}" \
  --ports=80
```

### 8. Add the backend to an existing URL map

If the global load balancer already exists, update the URL map instead of creating a new one.

Add a host rule and path matcher:

```shell
gcloud compute url-maps add-path-matcher EXISTING_URL_MAP_NAME \
  --global \
  --default-service=EXISTING_DEFAULT_BACKEND_SERVICE \
  --path-matcher-name="${SERVICE_NAME}-paths" \
  --new-hosts="app.example.com" \
  --backend-service-path-rules="/*=${BACKEND_SERVICE_NAME}"
```

To add a path rule to an existing path matcher, export the URL map, edit the YAML, validate it, and import it back. This is safer for production URL maps because you can review the full routing table before applying the change.

> **Warning:** URL map changes affect live traffic. Export and review the current URL map before modifying a production load balancer.

```shell
gcloud compute url-maps export EXISTING_URL_MAP_NAME \
  --destination=existing-url-map.yaml \
  --global

gcloud compute url-maps validate \
  --source=existing-url-map.yaml \
  --global

gcloud compute url-maps import EXISTING_URL_MAP_NAME \
  --source=existing-url-map.yaml \
  --global
```

## Validation

Check backend health:

```shell
gcloud compute backend-services get-health "${BACKEND_SERVICE_NAME}" \
  --global
```

Get the global IP address:

```shell
gcloud compute addresses describe "${GLOBAL_IP_NAME}" \
  --global \
  --format="value(address)"
```

Test the load balancer:

```shell
curl -i "http://$(gcloud compute addresses describe "${GLOBAL_IP_NAME}" --global --format='value(address)')/"
```

Check the Service, NEG annotation, and endpoints:

```shell
kubectl describe service "${SERVICE_NAME}" --namespace="${NAMESPACE}"

gcloud compute network-endpoint-groups list \
  --filter="name=${NEG_NAME}"
```

## Troubleshooting

- `backend service has no healthy backends`: verify the health check path, port, firewall rule, Pod readiness, and that endpoints exist in the zonal NEG.
- `NEG is empty`: verify the Service selector matches Pod labels and the Pods are Ready.
- `404 from load balancer`: verify the URL map host rules, path matchers, and path rules point to the expected backend service.
- `502 or 503 from load balancer`: verify the backend protocol and service port match what the application serves.
- Traffic only reaches one zone: add the NEG from each zone that has endpoints to the backend service.

## Cleanup

> **Warning:** Cleanup commands delete load balancer resources. Do not run them against shared or production resources unless you verified they are no longer used.

Delete the global load balancer resources created by this runbook:

```shell
gcloud compute forwarding-rules delete "${FORWARDING_RULE_NAME}" --global
gcloud compute target-http-proxies delete "${TARGET_PROXY_NAME}"
gcloud compute url-maps delete "${URL_MAP_NAME}"
gcloud compute backend-services delete "${BACKEND_SERVICE_NAME}" --global
gcloud compute health-checks delete "${HEALTH_CHECK_NAME}" --global
gcloud compute addresses delete "${GLOBAL_IP_NAME}" --global
```

Delete the Kubernetes resources:

```shell
kubectl delete service "${SERVICE_NAME}" --namespace="${NAMESPACE}"
kubectl delete deployment "${APP_NAME}" --namespace="${NAMESPACE}"
```

## References

- [Standalone network endpoint groups in GKE](https://cloud.google.com/kubernetes-engine/docs/how-to/standalone-neg)
- [External Application Load Balancer overview](https://cloud.google.com/load-balancing/docs/https)
- [Backend services and backends](https://cloud.google.com/load-balancing/docs/backend-service)
- [Health checks overview](https://cloud.google.com/load-balancing/docs/health-checks)
- [gcloud compute backend-services](https://cloud.google.com/sdk/gcloud/reference/compute/backend-services)
