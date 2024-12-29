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




