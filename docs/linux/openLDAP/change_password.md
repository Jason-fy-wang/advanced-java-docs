---
tags:
  - LDAP
  - LDAP-password
---

### 1. admin forget password
```shell
## step 1 ： generate new pwd
slappasswd


## step 2: config file:  change_pwd.ldif
dn: olcDatabase={1}hdb,cn=config
changetype: modify
replace: olcRootPW
olcRootPW: {SSHA}new_pwd_hash

ldapmodify -Y EXTERNAL -H ldapi:///  -f change_pwd.ldif

```


### 2. reset user password with admin account 
```shell

ldappassword -H ldap:///host  -x -D "cn=admin,dc=example,dc=com" -W -S "uid=username,ou=people,dc=example,dc=com"

-W: 提示admin password
-S: 提示输入新密码

```

### 3. reset admin password
```shell

ldappassword -H ldap:///host  -x -D "cn=admin,dc=example,dc=com" -W -a 旧密码 -S

```


