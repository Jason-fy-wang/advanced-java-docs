---
tags:
  - windows
  - proxy
  - v2RayN
  - TUN-mode
---

What is V2RayN TUN-Mode?  Whats function of it ?

> Tun-Mode will prompt proxy from application layer to system network forward.

> 开启TUN 后, 所有网络流量都会强制进入 V2Ray 核心处理, 应用程序及那个无法绕过代理.

```shell
application
	|
	|___  OS network stack
			|
			|__ __ TUN virtual network card
						|
						|_ _ _ V2Ray Core
								|
								|_ _ _ internation internet
```


V2RayN config:
* config sock proxy port is 7889(you can change this as you like)
* enable TUN
* set system proxy
* route set below.
![](./images/v2ray/v2ray.png)


after TUN enable, we can check TUN network connection in windows 11 as below:

control panel --> 网络和internet --> 网络和共享中心:

![](./images/v2ray/TUN.png)



