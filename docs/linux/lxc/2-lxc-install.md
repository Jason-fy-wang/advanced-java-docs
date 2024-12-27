---
tags:
  - lxc-install
  - lxc
  - install
---
> lxc install on centos8

```shell
yum install lxc lxc-libs lxc-templates lxc-devel

## cgroup manage command tools
yum install  libcgroup libcgroup-tools

# bridge manage tools
yum install bridge-utils 

## create rootfs tool
yum install debootstrap

## open switch for network
yum install centos-release-nfv-openvswitch
yum install  openvswitch-switch

```


> install from source

```shell

## clone source code
git clone 

## build package via make

```
