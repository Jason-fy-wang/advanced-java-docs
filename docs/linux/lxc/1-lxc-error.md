---
tags:
  - lxc-error
  - lxc
  - error
---
> create lxc container error:

```shell
lxc-create -t download -n c1 --logfile=/tmp/lxc-create.log --logpriority=DEBUG
Setting up the GPG keyring
ERROR: Unable to fetch GPG key from keyserver
lxc-create: c1: lxccontainer.c: create_run_template: 1625 Failed to create container from template
lxc-create: c1: tools/lxc_create.c: main: 331 Failed to create container c1
```

```shell
报错实在设置 GPG keyring 时,  检查时默认的 keyserver不可用
替换为如下:

lxc-create -t download -- --keyserver hkp://keyserver.ubuntu.com:80
lxc-create -t download -n c3 -- --keyserver hkp://keys.gnupg.net:80
lxc-create -t download -n c3 -- --keyserver hkp://pgp.mit.edu:80

```







