---
tags:
  - yum
  - repoaitory
  - source
---

```shell

wget http://mirrors.aliyun.com/repo/Centos-7.repo
yum install -y epel-release
# ali 的source instal
wget -O /etc/yum.repos.d/epel-7.repo http://mirrors.aliyun.com/repo/epel-7.repo


## centos 8 source:
curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-8.repo
```


