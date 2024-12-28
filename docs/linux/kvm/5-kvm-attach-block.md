---
tags:
  - kvm
  - attach-block
---

> attach block/device to kvm

```shell
# 1. create block
dd if=/dev/zero of=/tmp/new_disk.img bs=1M count=1024

# 2. attach device
virsh attach-disk kvm1 /tmp/new_disk.img vdb  --live
virsh domblklist kvm1
## login kvm and check
sudo fdisk -l
sudo dmesg | grep vdb

# 3. check xml
virsh dumpxml kvm1 | grep new_disk

# 4. detach device
virsh detach-disk kvm1 vdb --live


```


> attch block through attach-device

```shell

# 1. create xml for disk
vim disk.xml
<disk type='file' device='disk'>
<driver name='qemu' type='raw' cache='none'/>
<source file='/tmp/new_disk.img'/>
<target dev='vdb'/>
</disk>

# 2. attach device
virsh attach-device kvm1 --live disk.xml

# 3. check disk same as above

# 4. detach disk
virsh detach-device kvm1 --live disk.xml

```




