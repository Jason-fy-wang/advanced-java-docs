---
tags:
  - k8s
  - kubernetes
  - service
---
> when create service, how to debug whether service is health or available? 

1. check the service's endpoint is correct
	```shell
	kubectl get endpoints -A
	```
2. if endpoints is right, then check the service's selector label, which must be same as the pod's label.
3. k8s network is isolation net, so if we want to test service with curl, then we need jump into the k8s's net
	```shell
	kubectl run tmp --rm -it --image=busybox -- sh      # jump into tmp pod, which means we connect to k8s's net
	# wget -O-  http://mockservice.mockspace.svc.cluster.local/index.html
	```


>when the application port is inject from configmap, how to config the containerPort, livesness port , readingness port and service's target port ?

Leverage the named port.
```yaml
....
    spec:
      # initContainers:
        # Init containers are exactly like regular containers, except:
          # - Init containers always run to completion.
          # - Each init container must complete successfully before the next one starts.
      containers:
      - name:  mockapi
        image:  mock:1.0
        imagePullPolicy: IfNotPresent
        envFrom:
        - configMapRef:
            name: mockconfig
            #key: PORT    # we can inject specific key from configmap, but here we inject all keys
            #key: APP_ENV
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mocksecret
              key: DB_PASSWORD
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
          limits:
            cpu: 100m
            memory: 100Mi
        livenessProbe:
          tcpSocket:
            port: mockport   # use named port here.
          initialDelaySeconds: 5
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 3
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: mockport   # use named port here.
          initialDelaySeconds: 5
          timeoutSeconds: 2
          successThreshold: 1
          failureThreshold: 3
          periodSeconds: 10
        ports:
        - containerPort:  8080   # this doesn't affect the real pod port. just metadata. but it should be same as the real pod port
          name:  mockport   # named port
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mockservice
  namespace: mockspace
spec:
  selector:
    app: mockapi
  type: ClusterIP
  sessionAffinity: None
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800
  ports:
  - name: mockservice
    protocol: TCP
    port: 80
    targetPort: mockport    # use the named port
```



>how to check logs for pod ?

```shell
kubectl logs -f podname
```

>how to check ingress log?

```shell
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller -f
```



