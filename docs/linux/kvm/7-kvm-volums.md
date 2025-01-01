---
tags:
  - kvm
  - volums
---
 volume operation
```shell
# 1. list volume
virsh vol-list file_virtimages

# 2. create image
virsh vol-create-as file_virtimages new_volume.img 5G --format raw

# 3. image info
virsh vol-info /tmp/pool/new_image.img
virsh vol-info new_image.img --pool file_virtimages
qemu-img info /tmp/pool/new_image.img

# 4. dump volume xml
virsh vol-dumpxml new_image.img --pool file_virtimages

# 5. resize file
virsh vol-resize new_image.img --pool file_virtimages 2G
## reduce size
virsh vol-resize new_image.img --pool file_virtimages --shrink 1G

# 6. clone the image
virsh vol-clone new_image.img cloned.img --pool file_virtimages

# 7. delete image
virsh vol-delete cloned.img --pool file_virtimages

```