---
tags:
  - ubuntu-24
  - snap
  - snapd
  - systemctl
  - system-service
---

When i try to download ghostty through `snap`, get below error:
```
error: cannot install "ghostty": Post "https://api.snapcraft.io/v2/snaps/refresh": proxyconnect tcp: dial tcp 192.168.10.1:7890: connect: connection refused
```

Obviously, the issue is caused by the proxy configuration.  But the problem is the server don't config any proxy now. (Use proxy befoe, but not removed)

### 1. Check environment 

```shell
env | grep -i proxy
```

If get any proxy config from above command, that means the proxy is configed in somewhere.


### 2. Check apt proxy

```shell
cd /etc/apt/apt.conf.d

grep -i proxy *

# the file 90curtin-aptproxy contains the proxy
# 90curtin-aptproxy
Acquire::http::Proxy "http://192.168.20.1:7890";
Acquire::https::Proxy "http://192.168.20.1:7890";

```


Ok. comment above proxy config.


### 3. Check snapd service environment

```shell

systemctl show snapd | grep -i proxy

# serice config file
# /etc/systemd/system/snapd.service.d
rm -f /etc/systemd/system/snapd.service.d/snap_proxy.conf

# restart service
systemctl daemon-reload
systemctl restart snapd
```

Remove above proxy config file.


### 4. Check snapd proxy internally

```shell
snap get system proxy

# if exists, remove it 
sudo snap set system proxy.http=""
sudo snap set system proxy.https=""
```



