---
tags:
  - ubuntu
  - kvm
  - install
  - software
---
Ubuntu-24 install KVM

```shell
# check virtual support
zgrep -c '(vmx|svm)' | /proc/cpuinfo

# install kvm
sudo apt update
sudo apt install -y qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils virt-manager

# qemu-kvm  虚拟化核心模块
# libvirt*  管理虚拟机
# virt-manager  GUI 管理虚拟机
# brigde-utils  网络桥接支持

# check install
lsmod | grep kvm

# check use group
groups $USER

# append use to libvirt and kvm group
sudo usermod -aG libvirt  $USER
sudo usermod -aG kvm $USER

# check kvm 
virsh list --all

# gui create vm
virt-manager

# command install kvm
virt-install \
  --name ubuntu-vm \
  --ram 2048 \
  --vcpus=2 \
  --disk path=/var/lib/libvirt/images/ubuntu-vm.qcow2,size=10 \
  --os-type linux \
  --os-variant ubuntu22.04 \
  --cdrom=/path/to/ubuntu.iso \
  --network network=default \
  --graphics vnc


```






