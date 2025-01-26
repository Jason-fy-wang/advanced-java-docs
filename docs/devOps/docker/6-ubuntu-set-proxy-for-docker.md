---
tags:
  - docker
  - proxy
---

最近搭建k8s 集群, 那么所需要的哪些image呢, 就需要从 docker hub上下载. 
所以需要为docker 配置一个proxy, 让其可以访问到 docker hub

```shell
## ubuntu24
### 如何路径的配置, 之后重启即可.
cat /etc/systemd/system/docker.service.d/http-proxy.conf 
[Service]
Environment="HTTP_PROXY=http://192.168.10.1:7890"
Environment="HTTPS_PROXY=http://192.168.10.1:7890"
Environment="NO_PROXY=localhost,127.0.0.1,::1"

```





