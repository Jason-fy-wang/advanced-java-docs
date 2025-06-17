---
tags:
  - git
  - sort-tag
  - sort
  - tag
---
针对git中的tag进行查询时, 如何进行一些排序呢?   本篇总结一下git tag的排序操作
```shell
## list all tag
git tag 


## list all tag by ascending semver number
git tag --sort=v:refname

## list all reg by descending semver number
git tag --sort=-v:refname

## sort tag by lexicographically (alphabetic order)
git tag --sort=refname

## reverse alphabetic order
git tag --sort=-refname

## sort tag by the date of the commit
git tag --sort=committerdate

## reverse commit date
git tag --sort=-committerdate

## sort tag by `date of the tag created`.  Annotated tag only. because only annotated tag have the metadata like tagger's name, email and timestamp
git tag --sort=taggerdate
git tag --sort=-taggerdate

## sort by create date and specific the output format
git for-each-ref --sort=creatordate --format '%(refname:short) %(creatordate)' refs/tags



```







