---
tags:
  - kvm
  - kvm-clone
---

clone 一个新的kvm
```shell

# 方法一: 
# clone vm
## 需要先把 vm shutdown
virt-clone --original existing-kvm --name new_kvm_name --file /path/new-vm.qcow2

--original:  要复制的机(其必须是运行中)
--name:  新机名字
--file:  新机的磁盘存储路径


## 方法二
### 此方法复制的VM,由于是完全一样,所以需要修改hostname, 不然 动态获取DHCP IP 时会失败
### 新通过磁盘复制,之后安装
virsh vol-clone old new pool_name

### 桥接
virt-install --name new-vm --memory 2048 --vcpus 2 --disk path=/path/new-vm.qcow2,format=qcow2,bus=virtio --os-type linux --os-variant ubuntu24 --network bridge=virbr0,model=virtio --graphics none --import

### NAT
virt-install --name new-vm --memory 2048 --vcpus 2 --disk path=/path/new-vm.qcow2,format=qcow2,bus=virtio --os-type linux --os-variant ubuntu24 --network network=default,model=virtio --graphics none --import

### 通过此方法复制的 vm, 需要修改新的vm的hostname. 
#### 进入vm后, 修改 /etc/hostname 就可以

```


