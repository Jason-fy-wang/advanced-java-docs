---
tags:
  - k8s
  - kubernetes
  - ingress
---
> when request backend service through ingress, but get 400, how to debug this issue?

1. check the ingress log
```shell
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller -f
```
2. when we send request to ingress, the hostname should be same as the ingress's config
f.g  ingress config as below:
```yaml
spec:
  rules:
  - host: mock.service    # ingress's host
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: mockservice
            port:
              number: 80
```
so when we request to ingress, we should use:
```shell
curl -vv http://mock.service/index.html

#or

curl -H "Host: mock.service"  --url http://hostname/index.html
```

3. Verify the service is good [[6-debug-service]]
4. Verify pods are all good.
