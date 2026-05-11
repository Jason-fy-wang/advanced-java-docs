---
tags:
  - network
  - network-load-balance
  - load-balance
  - GCP
---

Record the GCP network load balancer create step:

```shell
# config region
gcloud config set compute/region us-west1
gcloud config set compute/zone us-west1-a
ZONE=$(gcloud config get-value compute/zone)
REGION=$(gcloud config get-value compute/region)
cat > startup.sh << EOF
#!/bin/bash
apt-get update
apt-get install apache2 -y
hostname=\$(curl -H "Metadata-Flavor: Google" http://169.254.169.254/computeMetadata/v1/instance/name)
echo "<h3>Web Server: \$hostname</h3>" | tee /var/www/html/index.html
service apache2 restart
EOF

# instances
gcloud compute instances create www1 \
		--zone=$ZONE  \
		--region=$REGION  \
		--tags=network-lb-tag  \
		--machine-type=e2-small \
		--image-family=debian-11 \
		--image-project=debian-cloud \
		--metadata-from-file startup-script=startup.sh

gcloud compute instances create www2 \
		--zone=$ZONE  \
		--region=$REGION  \
		--tags=network-lb-tag  \
		--machine-type=e2-small \
		--image-family=debian-11 \
		--image-project=debian-cloud \
		--metadata-from-file startup-script=startup.sh
		
gcloud compute instances create www3 \
		--zone=$ZONE  \
		--region=$REGION  \
		--tags=network-lb-tag  \
		--machine-type=e2-small \
		--image-family=debian-11 \
		--image-project=debian-cloud \
		--metadata-from-file startup-script=startup.sh
		
gcloud compute firewall-rules create www-firewall-network-lb --target-tags network-lb-tag --allow tcp:80
gcloud compute instances list

# config load balance service
## external ip
gcloud compute addresses create network-ln-ip-1 --region $REGION
gcloud compute addresses list

## health check
gcloud compute http-health-checks create basic-check --port=80 --request-path / --check-interval 10s --timeout 5s --healthy-threshold 2 --unhealth-threshold 2

## target pool and forwarding rule
gcloud compute target-pools create www-pool --region $REGION --http-health-check basic-check

gcloud compute target-pools add-instances www-pool --instances www1,www2,www3 --instances-zone $ZONE

gcloud compute forwarding-rules create www-rule --region $REGION --ports 80 --address network-lb-ip-1 --target-pool www-pool

gcloud compute forwarding-rules describe www-rule --region $REGION

gcloud compute forwarding-rules list --filter="name=www-rule" --format="json" 

```



