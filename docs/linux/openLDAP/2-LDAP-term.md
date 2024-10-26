---
tags:
  - LDAP
  - term
---

| key word | full name          | the mean of the word                                                                            |
| -------- | ------------------ | ----------------------------------------------------------------------------------------------- |
| dc       | domain Component   | 域名的部分, 其格式是将完整的域名分为几部分, 如: example.com, 变为: dc=example,dc=com                                   |
| uid      | User Id            | 用户ID, 如: 'Tom'                                                                                  |
| ou       | Organization Unit  | 组织单位, 类似于Linux中的子目录. 它是一个容器对象, 组织单位可以包含其他对象 (包括其他组织单元), 如:'market'                              |
| cn       | Common Name        | 如: "thoms Johanson"                                                                             |
| sn       | Suername           | 姓                                                                                               |
| dn       | Distinguished Name | 唯一辨别名. 类似于Linux文件系统中的绝对路径. 每个对象都有一个唯一的名称, 如"uid=tom,ou=market,dc=example,dc=com", 在一个目录中DN总是唯一的 |
| rdn      | Relative dn        | 相对辨别名称。 类似于Linux中的相对路径. 它是与目录结构无关的部分.   如:"uid=tom" 或 "cn=Thomas,Johansson"                     |
| c        | Country            | 国家. 如: CN US                                                                                    |
| o        | Organization       | 组织名. 如: example.inc                                                                             |


