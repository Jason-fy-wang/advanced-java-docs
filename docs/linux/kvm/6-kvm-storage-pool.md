---
tags:
  - kvm
  - storage-pool
---
KVM 支持多种类型的 storage-pool.  本文创建一个 `directory backend`的 storage pool.

```shell
# supported storage pool
Directory backend
Local filesystem backend
network filesystem backend
Logical backend
disk backend
iSCSI backend
SCSI backend
Multipath backend
RADOS blokc device backend
Sheepdog backend
Gluster backend
ZFS backend
Virtuozzo storage backend

```

> create directory backend storage pool

```shell
## 1. xml define
<pool type='dir'>
<name>file_virtimages</name>
<target>
	<path>/tmp/pool</path>
</target>
</pool>

# 2. define pool
virsh pool-define pool.xml

# 3. check all pool
virsh pool-list --all

# 4. start pool
virsh pool-start file_virtimages

# 5. autostart pool
virsh pool-autostart file_virtimages

# 6. check pool info
virsh pool-info file_virtimages

# 7. list the volume in pool
virsh vol-list file_virtimages

# 8. list volume info
virsh vol-info file_virtimages/image.img

# 9. we can install system on the volume 
virt-install --name kvm1 --ram 2048 --graphics vnc,listen=146.20.141.158  --hvm --disk vol=file_virtimages/image.img  --import

--import: 表示导入磁盘中的镜像, 而不是安装镜像到磁盘中. 常用于 migration操作

# 10. delete pool
virsh pool-destroy file_virtimages
virsh pool-undefine file_virtimages


```







