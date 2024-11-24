---
tags:
  - kernel
  - switch-kernel
  - centos8
---

```shell
## list available kernel
yum list kernel --showduplicates

## install kernel
yum install kernel kernel-4.18.0-348.7.1.el8_5

## installed kernel
grubby --info=ALL | egrep '^kernel'

## swtich kernel
grubby --set-default "/boot/vmlinuz-4.18.0-348.7.1.el8_5.x86_64"

## check detault kernel
grubby --default-kernel

## remove others kernel
yum remove kernel kernel-4.18.0-553.6.1.el8.x86_64
```

