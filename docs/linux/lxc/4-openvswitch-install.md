---
tags:
  - openvswitch
  - install
---
>centos stream 8 install openvswitch from source

```shell
# 1. download source 
wget https://www.openvswitch.org/releases/openvswitch-3.3.3.tar.gz
tar -xvf openvswitch-3.3.3.tar.gz && cd openvswitch-3.3.3

# 2. install build tools
dnf group install "Development Tools"

# 3. config
./boot.sh
./configure --prefix=/usr --sysconfdir=/etc 
./configure --prefix=/usr --localstatedir=/var --sysconfdir=/etc
make -j 2
make install

# 4. check installation
ovs-vsctl --version

# 5. init db
mkdir -p /var/run/openvswitch
ovsdb-tool create /etc/openvswitch/conf.db vswitchd/vswitch.ovsschema


# 6. start process
ovsdb-server --remote=punix:/var/run/openvswitch/db.sock     --remote=db:Open_vSwitch,Open_vSwitch,manager_options --pidfile --detach --log-file

ovs-vsctl --no-wait init
ovs-vswitchd --pidfile --**detach**

# 7. create service
vim /etc/systemd/system/openvswitch.service

[Unit]
Description=Open vSwitch
After=network.target

[Service]
ExecStart=/usr/bin/ovsdb-server --remote=punix:/var/run/openvswitch/db.sock     --remote=db:Open_vSwitch,Open_vSwitch,manager_options --pidfile --detach --log-file
ExecStartPost=/usr/bin/ovs-vswitchd --pidfile --detach
ExecStop=/usr/bin/ovs-appctl -t ovsdb-server exit
ExecStopPost=/usr/bin/ovs-appctl -t ovs-vswitchd exit
Restart=on-failure

[Install]
WantedBy=multi-user.target

```


```shell
debootstrap --arch=amd64 --include="openssh-server vim"  stable ./con1 http://httpredir.debian.org/debian
```


```shell
root@ubuntu:~# vim ~/config 
lxc.devttydir = lxc 
lxc.pts = 1024 
lxc.tty = 4 
lxc.pivotdir = lxc_putold 
lxc.cgroup.devices.deny = a 
lxc.cgroup.devices.allow = c *:* m 
lxc.cgroup.devices.allow = b *:* m 
lxc.cgroup.devices.allow = c 1:3 rwm 
lxc.cgroup.devices.allow = c 1:5 rwm 
lxc.cgroup.devices.allow = c 1:7 rwm 
lxc.cgroup.devices.allow = c 5:0 rwm 
lxc.cgroup.devices.allow = c 5:1 rwm 
lxc.cgroup.devices.allow = c 5:2 rwm 
lxc.cgroup.devices.allow = c 1:8 rwm 
lxc.cgroup.devices.allow = c 1:9 rwm 
lxc.cgroup.devices.allow = c 136:* rwm 
lxc.cgroup.devices.allow = c 10:229 rwm 
lxc.cgroup.devices.allow = c 254:0 rm 
lxc.cgroup.devices.allow = c 10:200 rwm 
lxc.mount.auto = cgroup:mixed proc:mixed sys:mixed 
lxc.mount.entry = /sys/fs/fuse/connections sys/fs/fuse/connections none bind,optional 0 0 
lxc.mount.entry = /sys/kernel/debug sys/kernel/debug none bind,optional 0 0 
lxc.mount.entry = /sys/kernel/security sys/kernel/security none bind,optional 0 0 
lxc.mount.entry = /sys/fs/pstore sys/fs/pstore none bind,optional 0 0 
lxc.mount.entry = mqueue dev/mqueue mqueue rw,relatime,create=dir,optional 0 0 
# Container specific configuration 
lxc.arch = x86_64 
lxc.rootfs = /root/container lxc.rootfs.backend = dir lxc.utsname = manual_container 
# Network configuration 
lxc.network.type = veth 
lxc.network.link = lxcbr0 
lxc.network.flags = up 
lxc.network.hwaddr = 00:16:3e:e4:68:91 
lxc.network.ipv4 = 10.0.3.151/24 10.0.3.255 
lxc.network.ipv4.gateway = 10.0.3.1 
# Limit the container memory to 512MB 
lxc.cgroup.memory.limit_in_bytes = 536870912
```


