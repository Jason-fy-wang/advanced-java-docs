---
tags:
  - kvm
  - kvm-secrets
---

```shell
# 1. define xml
<secret ephemeral='yes' private='yes'>
	<usage type='ceph'>
		<name>client.admin</name>
	</usage>
</secret>

# 2. define secret
virsh secret-define secret.xml

## list secret
virsh secret-list 
virsh secret-dumpxml SECRET-UUID

# 3. set value for secret
virsh secret-set-value UUID $(echo "password" | base64)

# 4. get secret value
virsh secret-get-value secret-uuid

# 5. use secret
<disk type='network' device='disk'>
  <driver name='qemu' type='raw'/>
  <auth username='admin'>
    <secret type='ceph' uuid='SECRET-UIID'/>
  </auth>
  <source protocol='rbd' name='pool/volume'>
    <host name='ceph-mon1' port='6789'/>
    <host name='ceph-mon2' port='6789'/>
  </source>
</disk>



# 6. delete secret
virsh secret-undefine secret.xml


```

