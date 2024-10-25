---
tags:
  - LDAP
  - operation
---
#### ldapseach
```shell
syntax:
ldapsearch [options] [filter]  [attributes]
## search all
ldapseach -x -d "dc=my-domain,dc=com"

## uid filter. 查询 jdoe条目
ldapsearch -x -d "dc=example,dc=com" "(uid=jdoe)"

## filter with attributes. 展示 jdoe的 cn mail属性
ldapsearch -x -d "dc=example,dc=com" "(uid=jdoe)"  cn mail

## 使用管理员查询
ldapsearch -x -D "cn=admin,dc=exmpla,dc=com" -w admin-passwd -b "dc=example,dc=com" 

## 查询特定组织单元 下的所有用户
ldapsearch -x -b "ou=people,dc=example,dc=com" "objectClass=person"

## limit return number
ldapsearch -x -b "dc=example,dc=com" -z 5 "objectClass=person"

## diaplay DN
ldapsearch -x -LLL -b "dc=example,dc=com" "(uid=jdoe)"  dn

## search group
ldapsearch -x -b "dc=example,dc=com" "(ou=group)"

## 查询 db文件
ldapsearch -Y EXTERNAL -H ldapi:/// -b "olcDatabase={2}hdb,cn=config" "(objectClass=olcDatabaseConfig)"

## 查询所有的 config
### 使用此 EXTERNAL方式, 是因为 "cn=admin,cn=config" 用户还没有创建
ldapsearch -Y EXTERNAL -H ldapi:/// -b "cn=config"
ldapsearch -x -D "cn=admin,cn=config" -W -b "cn=config"

## 查询所有加载的schema
### 使用此 EXTERNAL方式, 是因为 "cn=admin,cn=config" 用户还没有创建
ldapsearch -Y EXTERNAL -H ldapi:/// -b "cn=schema,cn=config"
## search with admin
ldapsearch -x -D "cn=admin,cn=config" -W -H ldapi:/// -b "cn=schema,cn=config"  objectClass

## search specific schema
ldapsearch -x -D "cn=admin,cn=config" -H ldapi:/// -W -b "cn=schema,cn=config" "cn={3}inetorgperson"
```

> olc:  openLdap configuration
> ldif:  LDAP input format (LDIF)

#### ldapmodify
```shell
## 修改rootCN
### modify-rootcn.ldif
dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcSuffix
olcSuffix: dc=example,dc=com

dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcRootDN
olcRootDN: cn=admin,dc=example,dc=com


ldapmodify -Y EXTERNAL -H ldapi:///  -f modify-rootcn.ldif

-Y EXTERNAL:   使用unix套接字
-H ldapi:/// :  使用本地套接字


### add passward
####1. generate password with slappasswd ({SSHA}TabEfVk2CdYj/PxBrf9zwjwMMkh/y+s8) loongson
####2. add passswd into config 
dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcRootPW
olcRootPW: {SSHA}TabEfVk2CdYj/PxBrf9zwjwMMkh/y+s8

ldapmodify -Y EXTERNAL -H ldapi:/// -f modify-pwd.ldif

### verify pwd
ldapsearch -x -D "cn=admin,dc=example,dc=com" -H ldapi:/// -W "ou=person"


### 从group中删除用户
dn: cn=g-admin,ou=group,dc=example,dc=com
changetype: modify
delete: member
member: cn=ryan,ou=backend,ou=tech,ou=person,dc=example,dc=com



```

#### ldapadd
```shell
### add group
### add people group
dn: dc=example,dc=com
objectClass: top
objectClass: dcObject
objectClass: organization
o: example Organization
dc: example

dn: ou=group,dc=example,dc=com
objectClass: organizationalUnit
ou: group
description: Organizational Unit for groups

dn: ou=person,dc=example,dc=com
objectClass: organizationalUnit
ou: person
description: Organizational Unit for persons


dn: Distinguished Name
ou: 单元名称

## verify
ldapsearch -x -H ldapi:/// -b "dc=example,dc=com"
### add uid=mike
dn: uid=mike,ou=person,dc=example,dc=com
objectClass: inetOrgPerson
cn: mike Doe
sn: Doe
uid: mike
mail: mike@example.com
userPassword: {SSHA}COMdojJocVGG14nMwtBAYmeVGy0+Kw3a

```

```shell
### 添加 cn=admin,cn=config
dn: olcDatabase={0}config,cn=config
changetype: modify
replace: olcRootDN
olcRootDN: cn=admin,cn=config
-
replace: olcRootPW
olcRootPW: {SSHA}97SYCvVs2k+2OC65EHDG3LDhrZhF1xLu

ldapadd -Y EXTERNAL -H ldapi:///  -f add-admin.ldif
```


```shell
## add schema
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/inetorgperson.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/cosine.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/nis.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/openldap.ldif
```


添加用户到组
```shell


```


#### ldapdelete

```shell


```

#### slaptest
```shell

```


#### ldappasswd
```shell
### update password
ldappasswd -x -H ldapi:/// -D "cn=mike,dc=example,dc=com" -w 123456 -s acdefg

```


#### 查询objectClass的属性
```shell
ldapsearch -x -D "cn=admin,cn=config" -H ldapi:/// -W -b "cn=schema,cn=config" "cn={3}inetorgperson"
```


#### module
```shell
### add group module
dc: cn=module,cn=config
cn: module
objectClass: olcModuleList
olcModulePath: /usr/lib64/openldap

dn: cn=module{0},cn=config
changetype: modify
add: olcModuleLoad
olcModuleLoad: memberof.la


ldapadd -Y EXTERNAL -H ldapi:///  -f add_module.ldif



### 添加 group
dn: olcOverlay=memberof,olcDatabase={2}hdb,cn=config
objectClass: olcConfig
objectClass: olcMemberOf
objectClass: olcOverlayConfig
objectClass: top
olcOverlay: memberof
olcMemberOfDangling: ignore
olcMemberOfRefInt: TRUE
olcMemberOfGroupOC: groupOfNames
olcMemberOfMemberAD: member
olcMemberOfMemberOfAd: memberOf

```


添加一个 group:
```shell
dn:  cn=g-admin,ou=group,dc=example,dc=com
objectClass: groupOfNames
cn: g-admin


```



> references
> [openldap](https://www.cnblogs.com/woshimrf/p/ldap.html)