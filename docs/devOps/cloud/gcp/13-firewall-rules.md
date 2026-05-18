---
tags:
  - GCP
  - firewall-rules
  - firewall
---
```shell
gcloud compute --project=$PROJECT firewall-rules create \ 
	manage-allow-icmp-ssh-rdp  \
	--direction=INGRESS \
	--priority=1000 \
	--network=managementnet \
	--action=ALLOW   \
	--rules=tcp:22,tcp:3389,icmp \
	--source-range=0.0.0.0/0
	

gcloud compute firewall-rules list --sort-by=NETWORK

gcloud compute firewall-rules delete manage-allow-icmp-ssh-rdp	

```
