---
tags:
  - GCP
  - load-balance
  - internal-load-balancer
---
GCP internal load balancer create steps:
```shell
ZONE=us-west1-a
REGION=us-west1

gcloud config set compute/zone $ZONE
gcloud config set sompute/region $REGION

cat > backend.sh << EOF
sudo chmod -R 777 /usr/local/sbin
sudo cat << PYTHON > /usr/local/sbin/servers.py
import http.server

def is_prime(a): return a!=1 and all(a % i for i in range(2,int(a**0.5)+1))

class myHandler(http.server.BasicHTTPRequestHandler):
	def do_GET(s):
		s.send_response(200)
		s.send_header("Content-type", "text/plain")
		s.end_headers()
		s.wfile.write(bytes(str(is_prime(int(s.path[1:]))).encode('UTF-8')))

http.server.HTTPServer(("", 80), myHandler).serve_forever()
PYTHON
nohup python3 /usr/local/sbin/servers.py >/dev/null 2>&1 &
EOF

gcloud compute instance-templates create primecalc \
	--metadata-from-file startup-script=backend.sh \
	--machine-type=e2-small \
	--no-address \
	--tags backend

gcloud compute firewall-rules create http --network default \ 
	--allow tcp:80  \
	--source-ranges $IP \   # only allow traffic from load-balance to backend instances
	

gcloud compute instance-groups managed create backend \
	--size 3 \
	--template primecalc \
	--zone $ZONE
	
gcloud compute health-check create http ilb-health --request-path /2

gcloud compute backend-services create prime-service \
	--load-balancing-schema internal \
	--region $REGION \
	--protocol tcp \
	--health-checks ilb-health
	
	
gcloud compute backend-service add-backend prime-service \
	--instance-group backend \
	--instance-group-zone $ZONE \
	--region $REGION

gcloud compute forwarding-rules create prime-lb \
	--load-balancing-schema internal \
	--region $REGION \
	--ports 80 \
	--address IP \
	--backend-service prime-service
	 
	

```

