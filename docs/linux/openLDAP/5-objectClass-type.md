---
tags:
  - openldap
  - objectclass
---
在openldap中每一个entry都会关联一个objectclass(role), 并以此设置对应的属性来表示此entry的内容.
在openldap中的objectclass 分为三种:
`structural`, `AUXILIARY`,`ABSTRUCT` and `OBSOLETE`.

```markdown
`structural` 是main class for entry.  每一个entry只有一个 structural的object.
不过通过继承, entry可以拥有多个objectClass的属性.

`Auxiliary` 作为辅助类型的object, 一个记录可以有多个 Auxiliary object.

`ABSTRUCT` 只用于 属性继承. 


# 继承 (只能单继承):
"( 2.5.6.0 NAME 'top' DESC 'top of the superclass chain' ABSTRACT MUST objectClass )"

"( 2.5.6.4 NAME 'organization' DESC 'RFC2256: an organization' SUP top STRUCTURAL MUST o MAY ( userPassword $ searchGuide $ seeAlso $ businessCategory $ x121Address $ registeredAddress $ destinationIndicator $ preferredDeliveryMethod $ telexNumber $ teletexTerminalIdentifier $ telephoneNumber $ internationaliSDNNumber $ facsimileTelephoneNumber $ street $ postOfficeBox $ postalCode $ postalAddress $ physicalDeliveryOfficeName $ st $ l $ description ) )"

organization继承 top.

"( 2.5.6.6 NAME 'person' DESC 'RFC2256: a person' SUP top STRUCTURAL MUST ( sn $ cn ) MAY ( userPassword $ telephoneNumber $ seeAlso $ description ) )"

"( 2.5.6.7 NAME 'organizationalPerson' DESC 'RFC2256: an organizational person' SUP person STRUCTURAL MAY ( title $ x121Address $ registeredAddress $ destinationIndicator $ preferredDeliveryMethod $ telexNumber $ teletexTerminalIdentifier $ telephoneNumber $ internationaliSDNNumber $ facsimileTelephoneNumber $ street $ postOfficeBox $ postalCode $ postalAddress $ physicalDeliveryOfficeName $ ou $ st $ l ) )"

organizationalPerson 继承 person.

```




