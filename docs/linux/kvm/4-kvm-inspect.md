---
tags:
  - kvm
  - inspect-info
project:
---
> **inspect kvm info**


> base info

```shell
#  id
virsh domid kvm1

# kvm base info
virsh dominfo kvm1

# kvm status
virsh domstate kvm1

# display connection status
virsh domcontrol ubuntu-22

# 

```

> CPU
```shell
# cpu 使用统计
virsh cpu-stats  kvm1

# cpu info
virsh vcpuinfo kvm1

# cpu count
virsh vcpucount kvm1

# host name
## 执行此操作,需要安装agent 到 kvm中
sudo apt install qemu-guest-agent
virsh domhostname kvm1

# io thread info
virsh iothreadinfo kvm1
virsh domjob info kvm1

# vnc info
virsh vncdisplay kvm1
virsh domdisplay kvm1


# screenshot
virsh screenshot kvm1

# set value
##  set vcpu number
virsh setvcpus kvm1
## attach/detch vcpu 
virsh setvcpu kvm1

# dump info 
viesh dumpxml kvm1

```

> memory

```shell
# meminfo
virsh domemstat kvm1

# set value
## set mem
virsh setmaxmem kvm1  --size 1024000
virsh setmem kvm1  --size 1049000
virsh memtune kvm1


```


> blk device

```shell

# block io info
virsh blkiotune kvm1

# list all block device
virsh domblklist kvm1

# check blk operation
virsh domblkstat kvm1
## check specific blk device stat
virsh domblkstat kvm1 sda --human

# display error
virsh domblkerror ubuntu-22

# blk device info
virsh domblkinfo ubuntu-22 vda

# 

```

> inet interface

```shell
# interface list
virsh domiflist kvm1


## check specific inet interface
virsh domifstat kvm1 vnet1

# up and down inet interface
virsh domif-setlink kvm1 vnet1 dowm
virsh domif-setlink kvm1 vnet1 up

# get link status
virsh domif-getlink kvm1 vnet1 

# tune interface
virsh domiftune kvm1 vnet1



```

> sched

```shell
# sched info
virsh schedinfo kvm1



```

>qemu-guest-agent

```shell
## 是安装到kvm中的agent
sudo apt install qemu-guest-agent

# test agent
virsh qemu-agent-command kvm1 '{"execute":"guest-ping"}'
virsh domhostname kvm1

```

