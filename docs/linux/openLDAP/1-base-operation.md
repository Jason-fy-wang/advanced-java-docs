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
> ldif:  LDAP data interchange format (LDIF)

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
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/cosine.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/nis.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/inetorgperson.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/openldap.ldif
```


add new entry
```shell
dn: ou=IT,ou=person,dc=example,dc=com
changetype: add
objectClass: organizationalUnit
ou: IT

dn: ou=backend,ou=IT,ou=person,dc=example,dc=com
changetype: add
objectClass: organizationUnit
ou: backend

dn: cn=mike,ou=backend,ou=IT,ou=person,dc=example,dc=com
changetype: add
objectClass: inetOrgPerson
cn: mike
departmentNumber: 1
sn: wang
title: sensor IT
mail: mike.wang@example.com
uid: 10000
displayName: MIKE


dn: cn=susan,ou=backend,ou=IT,ou=person,dc=example,dc=com
changetype: add
objectClass:  inetOrgPerson
cn: susan
departmentNumber: 1
sn: wang
title: java
mail: susan@example.com
uid: 10001
displayName: SUSAN

dn: ou=test,ou=IT,ou=person,dc=example,dc=com
changetype: add
objectClass: organizationalUnit
ou: test

dn: cn=tester,ou=test,ou=IT,ou=person,dc=example,dc=com
changetype: add
objectClass: inetOrgPerson
cn: tester
departmentNumber: 2
sn: wang
title: sensor tester
mail: tester@example.com
uid: 10002
displayName: TEST

dn: ou=HR,ou=person,dc=example,dc=com
changetype: add
objectClass: organizationalUnit
ou: HR

dn: cn=fang,ou=HR,ou=person,dc=example,dc=com
changetype: add
objectClass: inetOrgPerson
cn: fang
departmentNumber: 3
sn: huang
title: HRBP
mail: fang@example.com
uid: 10003
displayName: fang.huang


```

```shell
## 可用于登录 linux的账户
dn: uid=john,ou=person,dc=example,dc=com
changetype: add
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: organizationalPerson
uid: john
cn: John.Doe
sn: Doe
mail: john@example.com
userPassword: loongson
gidNumber: 1000
uidNumber: 10005
homeDirectory: /home/john

```

给上面的用户添加密码:
```shell
dn: cn=mike,ou=backend,ou=IT,ou=person,dc=example,dc=com
changetype: modify
replace: userPassword
userPassword: {SSHA}89fyIm6LLzMhcCYrOKme6JhtICq4LiCY

dn: cn=susan,ou=backend,ou=IT,ou=person,dc=example,dc=com
changetype: modify
replace: userPassword
userPassword: {SSHA}89fyIm6LLzMhcCYrOKme6JhtICq4LiCY

dn: cn=tester,ou=test,ou=IT,ou=person,dc=example,dc=com
changetype: modify
replace: userPassword
userPassword: {SSHA}89fyIm6LLzMhcCYrOKme6JhtICq4LiCY

dn: cn=fang,ou=HR,ou=person,dc=example,dc=com
changetype: modify
replace: userPassword
userPassword: {SSHA}89fyIm6LLzMhcCYrOKme6JhtICq4LiCY


ldapmodify -x -D "cn=admin,dc=example,dc=com" -W -f modify.ldif

```


#### ldapdelete

```shell
## 删除条目
ldapdelete -x -D "cn=admin,dc=example,dc=com" -W "cn=mike,ou=backend,ou=IT,ou=person,dc=example,dc=com"

ldapdelete -x -D "cn=admin,dc=example,dc=com" -W "ou=backend,ou=IT,ou=person,dc=example,dc=com"
```

#### slaptest
```shell
## verify 文件的完整性
slaptest -F /etc/openldap/slapd.d/
```


#### slapcat

```shell 
### backup 
slapcat -l export.ldif

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
dn: cn=module,cn=config
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
olcMemberOfMemberOfAD: memberOf

```


添加一个 group:
```shell
dn:  cn=g-admin,ou=group,dc=example,dc=com
objectClass: groupOfNames
cn: g-admin
member: cn=mike,ou=backend,ou=IT,ou=person,dc=example,dc=com
member: cn=fang,ou=HR,ou=person,dc=example,dc=com

# verify
ldapsearch -x -D "cn=admin,dc=example,dc=com" -W -b "ou=backend,ou=IT,ou=person,dc=example,dc=com" "(cn=mike)"  +

ldapsearch -x -D "cn=admin,dc=example,dc=com" -W -b "ou=person,dc=example,dc=com" "(|(cn=mike)(cn=fang))" memberOf

##
(|(cn=mike)(cn=fang))  : 或查询
```


#### ACL
```shell
syntax:
olcAccess: <access directive>
    <access directive> ::= to <what>
        [by <who> [<access>] [<control>] ]+
    <what> ::= * |
        [dn[.<basic-style>]=<regex> | dn.<scope-style>=<DN>]
        [filter=<ldapfilter>] [attrs=<attrlist>]
    <basic-style> ::= regex | exact
    <scope-style> ::= base | one | subtree | children
    <attrlist> ::= <attr> [val[.<basic-style>]=<regex>] | <attr> , <attrlist>
    <attr> ::= <attrname> | entry | children
    <who> ::= * | [anonymous | users | self
            | dn[.<basic-style>]=<regex> | dn.<scope-style>=<DN>]
        [dnattr=<attrname>]
        [group[/<objectclass>[/<attrname>][.<basic-style>]]=<regex>]
        [peername[.<basic-style>]=<regex>]
        [sockname[.<basic-style>]=<regex>]
        [domain[.<basic-style>]=<regex>]
        [sockurl[.<basic-style>]=<regex>]
        [set=<setspec>]
        [aci=<attrname>]
    <access> ::= [self]{<level>|<priv>}
    <level> ::= none | disclose | auth | compare | search | read | write | manage
    <priv> ::= {=|+|-}{m|w|r|s|c|x|d|0}+
    <control> ::= [stop | continue | break]


```


| Specifier | Entities                                        |
| --------- | ----------------------------------------------- |
| *         | All, including anonymous and authenicated users |
| anonymous | Anonymous (non-authenticated) Users             |
| users     | Authenticated users                             |
| self      | User associated with target entry               |
| dn[.]=    | Users matching a regular expression             |
| dh.=      | Users within scope of a DN                      |
|           |                                                 |


example:
```shell
access  to  <what>
		by  <who>  <access-level>
		by  <who>   <access-level>

#  example1: 密码自己可修改, person组可修改, anonymous 有认证权限; 其他人无权限
access  to  attrs=userpassword
		by self  write
		by anonymous  auth
		by group.exact="cn=person,ou=groups,dc=example,dc=com"   write
		by *   none

# example2:catlicense,homepostaladdress,homephone 属性 自己可修改.  hrperson组可修改. 其他无权限
access  to  attrs=catlicense,homepostaladdress,homephone
		by self   write
		by group.exact="cn=hrperson,ou=groups,dc=example,dc=com"  write
		by * none

# example3: self和person group可读写所有, user 只读. 其他无权限
access to *
	   by self    write
	   by group.exact="cn=person,ou=groups,dc=example,dc=com"  write
	   by users read
	   by *  none

# example4: 针对某一给条目下的subtree 只有管理员可访问
access to dn.subtree="ou=person,dc=example,dc=com"
	   by dn.exact="cn=admin,dc=example,dc=com"
	   by * none

# example 5:允许查看自己的条目
access to dn.children="ou=users,dc=example,dc=com"
	   by  self read
	   by *  none



```


| key word | access level  |
| -------- | ------------- |
| none     | 没有权限          |
| auth     | 允许认证          |
| compare  | 允许对属性进行比较     |
| search   | 允许搜索, 但不读取属性值 |
| read     | 允许读取条目        |
| write    | 允许写入条目或属性     |
| manage   | 允许管理权限(最高权限)  |


> references
> [openldap](https://www.cnblogs.com/woshimrf/p/ldap.html)
> [ldap](https://www.zytrax.com/books/ldap/)
   [ldap manual](https://www.openldap.org/doc/admin24/)
   [openLDAP](https://www.cnblogs.com/kevingrace/p/5773974.html)


