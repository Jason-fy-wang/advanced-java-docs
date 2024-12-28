---
tags:
  - kvm
  - console-login
---
> login with console

```shell 
## kvm add below config
  <console type='pty' tty='/dev/pts/1'>
      <source path='/dev/pts/1'/>
      <target type='serial' port='0'/>
      <alias name='serial0'/>
    </console>

```

```shell
# 在kvm 配置
## 1.  add blow config into /etc/default/grub
GRUB_CMDLINE_LINUX_DEFAULT="console=ttyS0,115200n8"
GRUB_TERMINAL="serial console"

## 2. update
sudo update-grub

## 3. start service
sudo systemctl start serial-getty@ttyS0.service


```


```shell
# login
virsh console kvm1
```