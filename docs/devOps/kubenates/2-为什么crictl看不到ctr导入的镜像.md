---
tags:
  - k8s
  - crictl
  - ctr
  - images
---
ctr 工具是有 namespace的概念的, 在进行导入时, 如何不指定namespace, 那么就导入`default` 默认 namespace.
crictl 去查询images时, 是查询的 `k8s.io` namespace , 故而会查不到.

> resolve:  如果是k8s的image, 那么就 在ctr导入时, 指定 namespace为k8s.
> `ctr --namespace k8s.io  images import img.tar.gz`











