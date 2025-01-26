---
tags:
  - kubernetes
  - error
---
> 新搭建好的k8s集群, 通过 kubectl访问时报错.

```shell
kubectl get nodes
E0126 12:55:23.205286    7477 memcache.go:265] couldn't get current server API group list: Get "https://192.168.122.3:6443/api?timeout=32s": net/http: TLS handshake timeout
E0126 12:55:33.219577    7477 memcache.go:265] couldn't get current server API group list: Get "https://192.168.122.3:6443/api?timeout=32s": net/http: TLS handshake timeout
E0126 12:55:43.229332    7477 memcache.go:265] couldn't get current server API group list: Get "https://192.168.122.3:6443/api?timeout=32s": net/http: TLS handshake timeout

# 原因:
## 使用了http_proxy, 导致请求通过proxy出去了.

## resolve
### 在 no_proxy 中设置 kubenates的网段,即 不进行代理
export no_proxy="localhost,127.0.0.1,192.168.20.20,192.168.122.0/24"



```







