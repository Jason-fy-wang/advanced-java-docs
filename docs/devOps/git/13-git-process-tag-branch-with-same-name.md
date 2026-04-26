---
tags:
  - git
  - same-git-branch
  - same-name
  - branch
  - tag
---
In current project,  we use the release version as remove branch and  release tag.  So which will cause the tag and branch have same name, and this cause some issue.


#### 1. how to delete tag and branch which have same name?

So how to delete the branch and tag that both of them have same name?

The answer is , we have to specific the tag and branch prefix to tell git we are going to delete tag or going to delete branch.

```shell
# delete local tag
git tag -d 'refs/tags/tagName'
## delete remote tag
git push --delete origin "refs/tag/tagname"

# delet branch
git branch -D 'refs/head/branchname'

## delete remote branch
git push origin  ":refs/heads/branchname"

git push --delete origin "refs/heads/branchName"

```


explain a bit about the format: 
 > why we can ignore --delete and set format: ':refs/head/branch' ??

Normally, we push change to remote as `git push local:remote`, f.g: 

```shell
# format 1
git push origin "refs/heads/main:refs/heads/main"

# with src only, which means remote is same branch with src
git push origin "refs/heads/main"

git push origin main:refs/heads/main

git push origin HEAD:refs/tags/v1.0

git push origin refs/heads/*:refs/heads/*

## exclude pattern  ^
### push all branches except for those starting with 'dev-'
git push origin refs/heads/*:^refs/heads/dev-*

# if src is empty, it delete <dst> branch
## delete branch dev
git push origin ":dev"



```





