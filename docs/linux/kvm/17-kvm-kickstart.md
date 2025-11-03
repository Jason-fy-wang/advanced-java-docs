---
tags:
  - kvm
  - kickstart
---
当通过ISO 镜像文件去安装kvm虚拟机(虚拟机一般是通过clone来生成新的主机, 更便捷, 快速),  或者是安装Linux主机系统时, 很多时候都需要交互输入, 那么有几百台或几千台时, 就很不方便了. 
不过Linux主机可以通过`kickstart`文件来实现自动化安装.

#### kickstart.cfg

ks.cfg 文件可以通过GUI生成:
```shell
yum install -y system-config-kickstart

# start server to generate kickstart file
system-config-kickstart
```

```text
## kickstart file example centos9-ks.cfg

#version=RHEL9
text
cdrom
eula --agreed

%addon com_redhat_kdump --enable --reserve-mb='auto'
%end

# Keyboard layouts
keyboard --xlayouts='us'

# System language
lang en_US.UTF-8

# Use CDROM installation media
cdrom

%packages
@^server-product-environment
@console-internet
@debugging
@performance
@system-tools
%end

# Run the Setup Agent on first boot
firstboot --enable

ignoredisk --only-use=/dev/vda
# System bootloader configuration
bootloader --append="crashkernel=1G-4G:192M,4G-64G:256M,64G-:512M" --location=mbr --boot-drive=vda
autopart --type=lvm
reboot --eject

# Partition clearing information
clearpart --none --initlabel

# System timezone
timezone America/New_York --utc
###########################################################################################

# 
# User Accounts
# Generate encrypted password: python -c 'import crypt; print(crypt.crypt("My Password"))'
# Or  openssl passwd -1 password
#
###########################################################################################
# Root password
rootpw --iscrypted <encrypted password>
# Admin user
user --groups=wheel --name=<default-user> --password=<encrypted password> --iscrypted --gecos="<default-user>"

# network
network --bootproto=dhcp --noipv6 --onboot=on --device=enp1s0

# SELinux
selinux --enforcing
```

#### kvm install
```shell
## kvm install

virt-install \
	--name centos9ks \
	--memory=2048M 
	--vcpus=2 
	--localtion Centos-stream-9-latest-x86_64-dvd1.iso  
	--disk size=20G  
	--network bridge=br0
	--graphics=none
	--os-variant=rhel9.0
	--console pty,target_type=serial
	--initrd-inject centos9-ks.cfg
	--extra-args "inst.ks=file://centos9-ks.cfg console=tty0 console=ttyS0,115200n8"

```

#### iso image
对于真实主机安装, 可以修改iso文件, 制作一个自动安装的镜像:
1. 拷贝 ks.cfg 到镜像文件中
2. 追加kernel参数
3. 重新制作镜像  

```shell
## kernel参数
append initrd=initrd.img inst.ks=http://10.32.5.1/mnt/archive/RHEL-8/8.x/x86_64/kickstarts/ks.cfg

kernel vmlinuz inst.ks=http://10.32.5.1/mnt/archive/RHEL-8/8.x/x86_64/kickstarts/ks.cfg
```

修改镜像中的 isolinux.cfg 文件:
```text
## centos6
## 挂在iso镜像文件到 isolinux目录上
[root@centos6 isolinux]# cat isolinux.cfg 
default vesamenu.c32
#prompt 1
timeout 600

display boot.msg

menu background splash.jpg
menu title Welcome to GETTOLIVE -CentOS 6.9-AUTO-IN! --> 这里可以自定义修改
menu color border 0 #ffffffff #00000000
menu color sel 7 #ffffffff #ff000000
menu color title 0 #ffffffff #00000000
menu color tabmsg 0 #ffffffff #00000000
menu color unsel 0 #ffffffff #00000000
menu color hotsel 0 #ff000000 #ffffffff
menu color hotkey 7 #ffffffff #ff000000
menu color scrollbar 0 #ffffffff #00000000
#==============源文件==================
label linux
  menu label ^Install or upgrade an existing system
  menu default
  kernel vmlinuz
  append initrd=initrd.img
label vesa
  menu label Install system with ^basic video driver
  kernel vmlinuz
  append initrd=initrd.img nomodeset
label rescue
  menu label ^Rescue installed system
  kernel vmlinuz
  append initrd=initrd.img rescue
label local
  menu label Boot from ^local drive
  localboot 0xffff
label memtest86
  menu label ^Memory test
  kernel memtest
  append -
  #============源文件结束================
  #============修改的内容================
label local
  menu label Boot from ^local drive
  localboot 0xffff　
  menu default
label mini
  menu label Install CentOS 6.9 ^Mini system
## 追加的参数
  kernel vmlinuz append initrd=initrd.img ks=cdrom:/isolinux/ks/ks6_mini.cfg
label desktop
  menu label Install CentOS 6.9 ^Desktop system
## 追加的参数
  kernel vmlinuz append initrd=initrd.img ks=cdrom:/isolinux/ks/ks6_desktop.cfg

```

```shell
# make iso image
[root@centos6 isolinux]# mkisofs -R -J -T -v --no-emul-boot --boot-load-size 4 --boot-info-table -V "CentOS 6.9 x86_64 boot" -b isolinux/isolinux.bin -c isolinux/boot.cat -o /root/boot.iso /tmp/auto-centos6/autoboot
```


[kickstart syntax reference](https://docs.redhat.com/en/documentation/Red_Hat_Enterprise_Linux/7/html/installation_guide/chap-kickstart-installations)
[redhat install with kickstart](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/8/html/system_design_guide/performing_an_automated_installation_using_kickstart#starting-kickstart-installations_system-design-guide)
[iso make](https://www.cnblogs.com/gettolive/p/9085380.html)

