---
tags:
  - ubuntu
  - install
  - k8s-install
---

### 1. 准备工作
```shell
# stop swap
sudo swapoff -a
sed -i '/ swap / s/^/#/' /etc/fstab

# 网络与防火墙
### 临时策略: 关闭
sudo systemctl disable --now ufw

### 上线 打开端口: 6443 2379-2380 10250 10251 10252 

# 内核参数和模块
cat <<EOF | sudo tee /etc/modules-load.d//k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

sudo systctl --system

```



### 2. 安装容器运行时(Containerd)
```shell
# 1. 安装依赖与配置镜像加速
sudo apt update
sudo apt install -y apt-transport-https ca-certificates curl
curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository \
  "deb [arch=amd64] https://mirrors.aliyun.com/docker-ce/linux/ubuntu \
  $(lsb_release -cs) stable"
sudo apt-get update



# 2. install containerd
sudo apt install -y containerd.io



# 3.配置镜像加速
在 `/etc/containerd/config.toml` 中找到或添加：
[plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
  endpoint = ["https://registry-1.docker.io"]
[plugins."io.containerd.grpc.v1.cri".registry.mirrors."k8s.gcr.io"]
  endpoint = ["https://registry.aliyuncs.com/google_containers"]

# containerd 默认配置
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1  
# cgroup 使用systemd
sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml

sudo systemctl restart containerd
```


### 3. install kubadm/kubelet/kubectl

```shell
# 1. add source
curl -s https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF
sudo apt-get update


## install
sudo apt-get install -y kubelet kubeadm kubectl
# free k8s version, which cause apt wont upgarade k8s version
sudo apt-mark hold kubelet kubeadm kubectl


```

```shell
# init firewall
##---------------------------- master and worker
### kubelet
sudo ufw allow 10250/tcp
### NodePort
sudo ufw allow 30000:32767/tcp

##----------------------------- master only
### kube-apiserver
sudo ufw allow 6443/tcp

### etcd
sudo ufw allow 2379:2380/tcp

### scheduler and controller-manage
sudo ufw allow 10251/tcp
sudo ufw allow 10252/tcp

# Calico (BGP)
sudo ufw allow 179/tcp

# Flannel VXLAN
sudo ufw allow 4789/udp

## allow ufw 转发
sudo sed -i 's/^DEFAULT_FORWARD_POLICY="DROP"/DEFAULT_FORWARD_POLICY="ACCEPT"/' /etc/default/ufw


sudo ufw status numbered

```

### 4.initial Master node
```shell
sudo kubeadm init \
  --image-repository registry.aliyuncs.com/google_containers \
  --kubernetes-version v1.27.3 \
  --pod-network-cidr=192.168.0.0/16

## 
--image-repository  指定阿里云加速


## config kubelet
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config


## re-create token
kubeadm token create --print-join-command \
  --image-repository registry.aliyuncs.com/google_containers

## debug error msg
journalctl -xeu kubelet
```

```shell
# 如果initial失败,则reset 重试
sudo kubeadm reset -f
```

### 5. add Worker nodes
```shell
sudo kubeadm join 10.0.0.1:6443 \
  --token abcdef.0123456789abcdef \
  --discovery-token-ca-cert-hash sha256:...

```

> note: 在添加work之后, 再去安装 net plugin

### 6. install Pod net plugin
```shell
kubectl apply -f https://mirrors.aliyun.com/kubernetes/calico/v3.25.0/calico.yaml
```


### 7. verify cluster
```shell
kubectl get nodes
kubectl get pods -A
```


