---
tags:
  - kvm
  - share-directory
---
> share directory between host and kvm

```shell
# 1. stop kvm
virsh stop kvm1

# 2. create folder
mkdir -p /tmp/shared
echo "shared folder" >> /tmp/shared/file

# 3. edit kvm xml
virsh edit kvm1

<devices>
...
<filesystem type='mount' accessmode='passthrough'>
	<source dir='/tmp/shared'/>
	<target dir='tmp_shared'/>
</filesystem>
...
</devices>

# 4. start kvm
virsh start kvm1

# 5. login kvm and mount the shared directory
mount -t 9p -o trans=virto tmp_shared /mnt



```


