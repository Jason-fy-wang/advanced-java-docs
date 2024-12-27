---
tags:
  - kvm
  - linux-optimize-for-kvm
---
> tune the kernel for low I/O latency

```shell
## 1. select correct IO schedule. available: noop, deadline, cfq
### check current config
cat /sys/block/sda/queue/scheduler

### change
echo 'GRUB_CMDLINE_LINUX="elevator=deadline"' >> /etc/default/grub
update-grub2

## 2. set guest vm I/O weight
virsh blkiotune --weight 100 ubuntu-22
### check config
virsh blkiotune --domain ubuntu-22

## 

```


> memory tune for kvm guest

```shell
## 1. increse huge page
sysctl vm.nr_hugepages=25000
### check
sysctl vm.nr_hugepages

## 2.check CPU if support 2M and 1G hugePage size
cat /proc/cpuinfo | grep -iE 'pse|pdpe1'

## 3. update GRUB to support 1G HugePage. add/update as below
GRUB_CMDLINE_LINUX_DEFAULT="rd.fstab=no acpi=noirq noapic cgroup_enable=memory swapaccount=1 quiet hugepagesz=1GB  hugepages=1"

update-grub
## 4. install hugePage
apt install hugepages

hugeadm --pool-list

sed -i 's/KVM_HUGEPAGES=0/KVM_HUGEPAGES=1/g'  /etc/default/qemu-kvm

## 5. mount hugeTable virtual filesystem

mkdir /hugepages
echo "hugetlbfs  /hugepages  hugetlbfs  mode=1770,gid=2021 0 0" >> /etc/fstab

mount -a

## 5. edit guest kvm to support hugepage
virsh edit kvm1

"""
<memoryBacking>
	<hugepages/>
</memoryBacking>
"""


## 6. tune guest kvm memory
### check memory info
virsh memtune kvm1

virsh memtune kvm1 --hard-limit 2G

```


> cpu preformance options

```shell
# 1. pin KVM to CPU 0 0
virsh vcpupin kvm1 0 0 --live
## pin KVM to CPU 1 0
virsh vcpupin kvm1 1 0 --live
## pin KVM to multiple cpu
virsh vcpupin ubuntu-22 0 0-7 --live

virsh vcpuinfo kvm1

# 2. schedule info
virsh schedinfo ubuntu-22
## set cpu weight
virsh schedinfo ubuntu-22 cpu_shares=2048
```



> NUMA tuning with libvirt

```shell
## NUMA supports following memory allocation policies.
strict: the placement will fail if memory cannot be allocated on the target node.

interleave: memory pages are allocated in a round-robin fashion.

preferred: this policy allows hypervisor to provide memory from other nodes. in case there is not enough memory available from the specified nodes.

## 1. install numactl
dnf install numactl

## 2. check kvm numa memory status
numastat -c ubuntu-22

## 3. set numa tune
virsh edit ubuntu-22

<vcpu placement="static">2<vcpu>
<numatune>
	<memory node='strict' nodeset='1'/>
</numatune>

## 4. check numatune set
virsh numatune ubuntu-22


### numa start
systemctl start numad

### numa rebalance log
numad -S -0 -p $(pidof ubuntu-22)
less /var/log/numad.log




```

> tune the kernel network performance

```shell
## 1. TCP send and receive socket buffer size
### check rev buffer value
sysctl net.core.rmem.max
sysctl net.core.wmem.max

### set value
sysctl net.core.rmem.max=33554432
sysctl net.core.wmem.max=33554432

## 2. TCP buffer limits: min default max
sysctl net.ipv4.tcp_rmem="4096 87380 33554432"
sysctl net.ipv4.tcp_wmem="4096 65536 33554432"

## 3. TCP window scaling enable (滑动窗口)
sysctl net.ipv4.tcp_window_scaling=1

## 4. increase length of transmit queue of network interface
ifconfig eth0 txqueuelen 5000

## 5. reduce tcp_fin_timeout value
systcl net.ipv4.tcp_fin_timeout=30

## 6. reduce tcp_keepalive_intval
sysctl net.ipv4.tcp_keepalive_intvl=30

## 7. enable fast recycling of TIME_WAIT sockets
sysctl net.ipv4.tcp_tw_recycle=1

## 8. enable reusing of sockets in TIME_WAIT state for new connection
sysctl net.ipv4.tcp_tw_reuse=1

## 9. pluggable congestion control algorithms
sysctl net.ipv4.tcp_available_congestion_control

## 10. if overwhelmed with SYN connection
sysctl net.ipv4.tcp_max_syn_backlog=16384
### server 端重发 syn-ack 响应的重试次数
sysctl net.ipv4.tcp_synack_retries=1 


## 11. increase max file descriptions
sysctl fs.file-max=10000000

### check file description current max and available number
sysctl fs.file-nr

## 12. if hypervisor using stateful iptable rules, the nf_conntrack kernel module might run out of memory for connection tracking and an error will be logged: nf_conntrack :table full, dropping packet.   In order to raise that limit and therefore allocate more memory, need to calculate how much RAM echo connection uses. You can get that information from /proc/slabinfo.  The nf_conntrack entry shows the active entries, how big echo object is. and how many fit in slab (echo slab fits in one or more kernel page, usually 4K if not using hugepages.).  Accouting for the overhead of the kernel page size, you can see from the slabinfo that echo nf_conntrack object takes about 316 bytes. So to track 1M connections, you will need to allocate roughly 316M memory.

sysctl net.netfilter.nf_conntrack_count
sysctl net.netfilter.nf_conntrack_max=1000000

# hashsize = nf_conntrack_max / 4
echo "250000" > /sys/module/nf_conntrack/parameters/hashsize





```






