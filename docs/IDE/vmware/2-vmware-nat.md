---
tags:
  - vmware
  - NAT
  - DHCP
---
> Vmware安装好 ubuntu24 虚拟机后, 发现如何不能上网, 网卡状态 显示 `NO-CARRIER`,同时也没有IP.

这是Vmware自带的NAT 和DHCP 服务不太正常, 打开windows中的 Vmware-NAT以及 Vmware-DHCP 服务进行重启.
![](./images/net/service.png)

![](./images/net/nat.png)








