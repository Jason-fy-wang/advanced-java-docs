---
tags:
  - frp
  - intranet
---

`frps` deploy on your public server,  and `frpc`deploy on your local.

```shell
# deploy frp
## config file: frp.ini
## open firewall rule for port: 7000
[common]
bind_port=7000


frps -c frp.ini

```


```shell
# deploy your frpc
## config file: frpc.ini
[common]
server_addr=public server ip
server_port=7000

[web]
type=http
local_port=5000
custom_domain=your domain name


frpc -c frpc.ini


```

