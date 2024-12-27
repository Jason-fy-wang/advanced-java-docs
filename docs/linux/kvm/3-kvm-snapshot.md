---
tags:
  - kvm
  - snapshot
---
> create snapshot for kvm

```shell
# create
virsh snapshot-create-as --domain ubuntu-22 --name initial-install

# list snapshot
virsh shapshot-list --domain ubuntu-22

# delete snapshot
virsh snapshot-delete --domain ubuntu-22 initial-install

# list snapshot info
virsh snapshot-info --domain ubuntu-22 initial-install


# snapshot rollback
virsh snapshot-revert  ubuntu-22 initial-install

# dump snapshot xml definenation
virsh snapshot-dumpxl  ubuntu-22 initial-install

# edit snapshot info
virsh snapshot-edit ubuntu-22 initial-install

# 当前正在使用的 snapshot
virsh snapshot-current ubuntu-22


```

