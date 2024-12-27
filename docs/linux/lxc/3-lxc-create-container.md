---
tags:
  - lxc-create
  - containerd
  - lxc
---
> create container

```shell
[root@localhost admin]# lxc-create -t download -n c3 -- --keyserver hkp://keyserver.ubuntu.com:80
Setting up the GPG keyring
Downloading the image index

---
DIST    RELEASE ARCH    VARIANT BUILD
---
almalinux       8       amd64   default 20241223_23:08
almalinux       8       arm64   default 20241223_23:08
almalinux       9       amd64   default 20241223_23:08
almalinux       9       arm64   default 20241223_23:08
alpine  3.18    amd64   default 20241223_13:00
alpine  3.18    arm64   default 20241223_13:00
alpine  3.18    armhf   default 20241223_13:00
alpine  3.19    amd64   default 20241223_13:00
alpine  3.19    arm64   default 20241223_13:00
alpine  3.19    armhf   default 20241223_13:00
alpine  3.20    amd64   default 20241223_13:00
alpine  3.20    arm64   default 20241223_13:00
alpine  3.20    armhf   default 20241223_13:02
alpine  3.21    amd64   default 20241223_13:00
alpine  3.21    arm64   default 20241223_13:00
alpine  3.21    armhf   default 20241223_13:00
alpine  edge    amd64   default 20241223_13:00
alpine  edge    arm64   default 20241223_13:00
alpine  edge    armhf   default 20241223_13:02
alt     Sisyphus        amd64   default 20241224_01:17
alt     Sisyphus        arm64   default 20241224_01:17
alt     p10     amd64   default 20241224_01:17
alt     p10     arm64   default 20241224_01:17
alt     p11     amd64   default 20241224_01:17
alt     p11     arm64   default 20241224_01:17
amazonlinux     2       amd64   default 20241224_05:09
amazonlinux     2       arm64   default 20241224_05:09
amazonlinux     2023    amd64   default 20241224_05:09
archlinux       current amd64   default 20241224_04:18
archlinux       current arm64   default 20241224_04:18
busybox 1.36.1  amd64   default 20241224_06:00
busybox 1.36.1  arm64   default 20241224_06:00
centos  9-Stream        amd64   default 20241224_07:08
centos  9-Stream        arm64   default 20241224_07:08
debian  bookworm        amd64   default 20241224_05:24
debian  bookworm        arm64   default 20241224_05:24
debian  bookworm        armhf   default 20241224_05:24
debian  bullseye        amd64   default 20241224_05:24
debian  bullseye        arm64   default 20241224_05:24
debian  bullseye        armhf   default 20241224_05:24
debian  buster  amd64   default 20241224_05:24
debian  buster  arm64   default 20241224_05:24
debian  buster  armhf   default 20241224_05:35
debian  trixie  amd64   default 20241224_05:24
debian  trixie  arm64   default 20241224_05:24
debian  trixie  riscv64 default 20241224_05:24
devuan  beowulf amd64   default 20241223_11:50
devuan  beowulf arm64   default 20241223_11:50
devuan  chimaera        amd64   default 20241223_11:50
devuan  chimaera        arm64   default 20241223_11:50
devuan  daedalus        amd64   default 20241223_11:50
devuan  daedalus        arm64   default 20241223_11:50
fedora  39      amd64   default 20241223_20:33
fedora  39      arm64   default 20241223_20:33
fedora  40      amd64   default 20241223_20:33
fedora  40      arm64   default 20241223_20:33
fedora  41      amd64   default 20241223_20:33
fedora  41      arm64   default 20241223_20:33
funtoo  next    amd64   default 20241223_16:45
kali    current amd64   default 20241223_17:14
kali    current arm64   default 20241223_17:14
mint    ulyana  amd64   default 20241224_08:51
mint    ulyssa  amd64   default 20241224_08:51
mint    uma     amd64   default 20241224_08:51
mint    una     amd64   default 20241224_08:51
mint    vanessa amd64   default 20241224_08:51
mint    vera    amd64   default 20241224_08:51
mint    victoria        amd64   default 20241224_08:51
mint    virginia        amd64   default 20241224_08:51
mint    wilma   amd64   default 20241224_08:51
nixos   24.05   amd64   default 20241223_01:02
nixos   24.05   arm64   default 20241223_01:02
nixos   24.11   amd64   default 20241223_01:01
nixos   24.11   arm64   default 20241223_01:00
nixos   unstable        amd64   default 20241223_01:00
nixos   unstable        arm64   default 20241223_01:01
openeuler       20.03   amd64   default 20241223_15:48
openeuler       20.03   arm64   default 20241223_15:48
openeuler       22.03   amd64   default 20241223_15:48
openeuler       22.03   arm64   default 20241223_15:48
openeuler       24.03   amd64   default 20241223_15:48
openeuler       24.03   arm64   default 20241223_15:48
openeuler       24.09   amd64   default 20241223_15:48
openeuler       24.09   arm64   default 20241223_15:48
opensuse        15.5    amd64   default 20241224_04:20
opensuse        15.5    arm64   default 20241224_04:20
opensuse        15.6    amd64   default 20241224_04:20
opensuse        15.6    arm64   default 20241224_04:20
opensuse        tumbleweed      amd64   default 20241224_04:42
opensuse        tumbleweed      arm64   default 20241224_04:20
openwrt 21.02   amd64   default 20241223_11:57
openwrt 21.02   arm64   default 20241223_11:57
openwrt 22.03   amd64   default 20241223_11:57
openwrt 22.03   arm64   default 20241223_11:57
openwrt 23.05   amd64   default 20241223_11:57
openwrt 23.05   arm64   default 20241223_11:57
openwrt snapshot        amd64   default 20241223_11:57
openwrt snapshot        arm64   default 20241223_11:57
oracle  7       amd64   default 20241224_07:46
oracle  7       arm64   default 20241224_07:58
oracle  8       amd64   default 20241224_07:46
oracle  8       arm64   default 20241224_08:19
oracle  9       amd64   default 20241224_07:46
oracle  9       arm64   default 20241224_07:56
plamo   7.x     amd64   default 20241224_01:33
plamo   8.x     amd64   default 20241224_01:33
rockylinux      8       amd64   default 20241224_02:06
rockylinux      8       arm64   default 20241224_02:06
rockylinux      9       amd64   default 20241224_02:06
rockylinux      9       arm64   default 20241224_02:06
slackware       15.0    amd64   default 20241223_23:08
slackware       current amd64   default 20241223_23:08
springdalelinux 7       amd64   default 20241224_06:38
springdalelinux 8       amd64   default 20241224_06:38
springdalelinux 9       amd64   default 20241224_06:38
ubuntu  focal   amd64   default 20241223_12:12
ubuntu  focal   arm64   default 20241223_12:12
ubuntu  focal   armhf   default 20241223_12:37
ubuntu  focal   riscv64 default 20241223_13:35
ubuntu  jammy   amd64   default 20241223_12:12
ubuntu  jammy   arm64   default 20241223_12:12
ubuntu  jammy   armhf   default 20241223_12:36
ubuntu  jammy   riscv64 default 20241223_14:01
ubuntu  noble   amd64   default 20241223_12:12
ubuntu  noble   arm64   default 20241223_12:12
ubuntu  noble   armhf   default 20241223_12:36
ubuntu  noble   riscv64 default 20241223_14:17
ubuntu  oracular        amd64   default 20241223_12:12
ubuntu  oracular        arm64   default 20241223_12:12
ubuntu  oracular        armhf   default 20241223_12:12
ubuntu  oracular        riscv64 default 20241223_13:18
voidlinux       current amd64   default 20241223_17:10
voidlinux       current arm64   default 20241223_17:10
---

Distribution: 
centos
Release: 
9-Stream
Architecture: 
amd64

Downloading the image index
Downloading the rootfs
Downloading the metadata
The image cache is now ready
Unpacking the rootfs

---
You just created a Centos 9-Stream x86_64 (20241224_07:08) container.
```

