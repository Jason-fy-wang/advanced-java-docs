---
tags:
  - error
  - jenkins-agent
  - jenkins
---
> ca certificate 证书位置改变导致的错误

在进行一个接口的访问时, 在内部通常是这样编写
```shell
curl --cacert /etc/ssl/certs/ca-certificates.crt  url paramters
```

在原来的agent 中, 使用的 ubuntu 的镜像, 其默认证书位置为: `/etc/ssl/certs/ca-certificates.crt`,   最新的镜像使用的是 redhat系统, 其默认证书的位置为: `/etc/ssl/certs/ca-bundle.crt`.

因为两个系统的默认证书不一致, 导致其参数中的指定的证书不能使用, 出现了 tls 握手错误.

解决:
```shell
## 删除指定的 cacert 参数及其value.

针对此情况, 此方法最适合.
1.本就是使用的默认证书, 不需要指定
2.如果使用的默认证书位置修改, 或者名称修改就会导致错误. 跟此次错误类似, 故可以在 镜像的系统中通过 `环境变量` 来修改默认证书位置.
3.除非使用特定的证书, 否则不该指定ca 证书
```






