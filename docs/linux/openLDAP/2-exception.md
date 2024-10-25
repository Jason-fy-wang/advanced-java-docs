---
tags:
  - exception
  - LDAP
---
### 添加entry时 error: ldap_add: No such object (32)
```shell
dn: ou=person,dc=example,dc=com
objectClass: organizationalUnit
ou: person
description: Organizational unit for person


[hosts@name2 ldap]# ldapadd -H ldapi:/// -x -D "cn=admin,dc=example,dc=com" -W -f add_person.ldif 
Enter LDAP Password: 
adding new entry "ou=person,dc=example,dc=com"
ldap_add: No such object (32)


此异常:
是parent entry not exists

创建parent entry

dn: dc=example,dc=com
objectClass: top
objectClass: domain
dc: example


```

