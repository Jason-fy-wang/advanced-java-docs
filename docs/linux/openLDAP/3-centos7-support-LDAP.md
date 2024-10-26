---
tags:
  - LDAP
  - centos7
  - Linux
  - Auth
---
在centos7上支持LDAP认证, 可以使用`sssd`和`nslcd`, 不过sssd的功能更强大, 在这里使用 sssd 搭建环境

#### 安装依赖包
```shell
sudo yum install -y authconfig sssd sssd-ldap oddjob-mkhomedir

authconfig: 用于配置身份源和方法
sssd: 负责与LDAP服务器进行身份验证
sssd-ldap: 提供SSSD和LDAP的支持
oddjob-mkhomedir: 用于自动创建用户的home目录
```



#### LDAP 配置
```shell
sudo authconfig --enableldap --enableldapauth \
    --ldapserver=ldap://ldap.example.com \
    --ldapbasedn="dc=example,dc=com" \
    --enablemkhomedir --update

```


#### sssd 配置

```shell
### sssd配置文件如下,由authconfig生成
[domain/default]
autofs_provider = ldap
cache_credentials = True
ldap_search_base = ou=IT,ou=person,dc=example,dc=com
id_provider = ldap
auth_provider = ldap
chpass_provider = ldap
ldap_uri = ldap://192.168.30.15
ldap_id_use_start_tls = False
ldap_tls_cacertdir = /etc/openldap/cacerts

[sssd]
services = nss, pam, autofs
config_file_version = 2
domains = default, LDAP

[domain/LDAP]
id_provider = ldap
auth_provider = ldap
ldap_uri = ldap://192.168.30.15
ldap_search_base = dc=example,dc=com
ldap_id_use_start_tls = false
cache_credentials = true
ldap_tls_reqcert = never

[pam]

[autofs]
# 如果使用的是 LDAPS
# ldap_uri = ldaps://ldap.example.com
# ldap_tls_reqcert = demand

chmod 600 /etc/sssd/sssd.conf 
chown root:root /etc/sssd/sssd.conf
```

启动sssd
```shell
systemctl start sssd
systemctl enable sssd
```


#### 配置NSS 和PAM
>NSS: Name Serice Switch
>PAM: Pluggable Auhentication Module

NSS 和PAM控制系统如何解析用户和组信息以及认证, 使用authconfig命令已经自动完成了配置,  但要确认 `/etc/nsswitch.conf`中的`passwd`,`group`,`shadow`字段包含了`sss`.

```shell
cat /etc/nsswitch.conf
...
passwd:     files sss ldap
shadow:     files sss ldap
group:      files sss ldap
...
```


#### 验证
```shell
id  LDAP-user
```