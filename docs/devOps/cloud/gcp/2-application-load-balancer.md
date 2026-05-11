---
tags:
  - application-load-balancer
  - GCP
  - load-balance
---

GCP application load balancer create steps:
```shell
ZONE="us-west1-a"
REGION="us-west1"

gcloud config set compute/region $REGION
gcloud config set compute/zone $ZONE

cat > startup.sh << EOF
#!/bin/bash
apt-get update
apt-get install apache2 -y
hostname=\$(curl -H "Metadata-Flavor: Google" http://169.254.169.254/computeMetadata/v1/instance/name)
echo "<h3>Web Server: \$hostname</h3>" | tee /var/www/html/index.html
systemctl restart apache2
EOF

# application-load-balancer
gcloud compute instance-templates create lb-backend-template \
	--region=$REGION \
	--zone=$ZONE  \
	--network=default \
	--subnet=default \
	--tags=allow-health-check \
	--machine-type=e2-medium \
	--image-family=debian-11 \
	--image-project=debian-cloud \
	--metadata-from-file startup-script=startup.sh

gcloud compute instance-groups managed create lb-backend-group \
	--template=lb-backend-template \
	--size=2 \
	--zone=$ZONE

## below 2 commands are optional cmd
gcloud compute instance-groups managed set-named-ports lb-backend-group  \
	--named-ports=http:80 \
	--zone=$ZONE
gcloud compute instance-groups unmanaged set-named-ports my-group \
	--named-ports=http:80 \
	--zone=$ZONE
	
gcloud compute firewall-rules create fw-allow-health-check  \
	--network=default \
	--action=ALLOW \
	--direction=INGRESS \
	--source-ranges=130.211.0.0/22,35.191.0.0/16 \
	--target-tags=allow-health-check \
	--rules=tcp:80

gcloud compute addresses create lb-ipv4-1 --ip-version=IPV4 --global

gcloud compute addresses lb-ipv4-1 --global --format="json"

gcloud compute health-checks creae http-basic-check \
	--port 80 \
	--request-path / \
	--check-interval 10s \
	--timeout 5s \
	--healthy-threshold 2 \
	--unhealthy-threshold 2

gcloud compute backend-services create web-backend-service \
	--protocol=HTTP \
	--port-name=http \
	--health-checks=http-basic-check \
	--global

gcloud compute backend-services add-backend web-backend-service \
	--instance-group=lb-backend-group \
	--instance-group-zone=$ZONE \
	--global

gcloud compute url-maps create web-map-http \
	--default-service web-backend-service

gcloud compute target-http-proxies create http-lb-proxy \
	--url-map web-map-http

gcloud compute forwarding-rules create http-content-rule \
	--address=lb-ipv4-1  \
	--global   \
	--target-http-proxy=http-lb-proxy \
	--ports=80


```

