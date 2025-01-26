---
tags:
  - kubernetes
  - k8s-install
---
下载kuberates images
```shell

## 方法一:  如何可以访问google registry. 直接使用下面方法.
# list needed images
kubeadm config images list

# 在init 之前先拉去 images
sudo kubeadm config images pull  --image-repository='registry.cn-hangzhou.aliyuncs.com/google_containers'

# init
sudo kubeadm init --image-repository='registry.cn-hangzhou.aliyuncs.com/google_containers'

# 失败回滚
sudo kubeadm reset


## 方法二: 不可直接访问google
### 为docker配置一个代理, 然后从 docker hub上提前下载好所有的镜像.
### 所需image 如下:
images="registry.k8s.io/kube-apiserver:v1.30.9 registry.k8s.io/kube-controller-manager:v1.30.9  registry.k8s.io/kube-scheduler:v1.30.9 registry.k8s.io/kube-proxy:v1.30.9  registry.k8s.io/pause:3.9  registry.k8s.io/etcd:3.5.15-0 calico/node:v3.29.0 calico/cni:v3.29.0 calico/kube-controllers:v3.29.0 registry.k8s.io/coredns/coedns:v1.11.3"

### 其中: kube-proxy  etcd   pause calico/node calico/cni coedns 需要在 controller 和 worker-node 同时存在

### 之后执行一下方法
sudo kubeadm init



```






> reference

[k8s install step](https://hbayraktar.medium.com/how-to-install-kubernetes-cluster-on-ubuntu-22-04-step-by-step-guide-7dbf7e8f5f99)